# K8s Service 类型本质归档

> 技术栈前提：CNI = Calico 纯 BGP 模式（L3 路由直通，无 VXLAN/IPIP 封装）；kube-proxy = IPVS 模式（rr 调度）。
> 示例 IP 均为通用地址，可套用到任意集群。

---

## 快速总览

| 类型 | 分配 ClusterIP | 创建 IPVS 规则 | kube-proxy 参与 | DNS 行为 | 典型场景 |
|------|--------------|--------------|----------------|---------|---------|
| ClusterIP | 是 | 是（ClusterIP 为 VIP） | 是 | A 记录 -> ClusterIP | 集群内部服务互访 |
| Headless | 否（None） | 否 | 否 | A 记录 -> 所有 Pod IP | StatefulSet、自定义 LB |
| NodePort | 是 | 是（两组规则） | 是 | 同 ClusterIP | 对外暴露，无云 LB |
| LoadBalancer | 是 | 是（继承 NodePort） | 是 | 同 ClusterIP | 云上 / MetalLB 对外暴露 |
| ExternalName | 否 | 否 | 否 | CNAME -> 外部域名 | 集群内访问外部服务别名 |

**类型层级关系：**

```
ClusterIP  ─────────────── 基础层（集群内 IPVS VIP）
  └── NodePort ─────────── ClusterIP + 每节点额外监听 nodeIP:Port
        └── LoadBalancer── NodePort + 外部 LB（云 / MetalLB）拨入

Headless ──────────────── 无 VIP，纯 DNS A 记录（直达 Pod IP）
ExternalName ──────────── 无 VIP，纯 DNS CNAME（指向外部域名）
```

---

## 1. ClusterIP

**本质**：在集群**每个节点**上创建 IPVS 虚拟服务器规则。ClusterIP 以 `/32` 绑定在 `kube-ipvs0` dummy 网卡上充当 VIP，后端 Pod IP 是 Real Server（RS），IPVS 内核哈希表 O(1) 做 DNAT 负载均衡。

- **kube-ipvs0 的作用**：使内核认为 ClusterIP 是"本机地址"，数据包命中后进入 `INPUT` 链，IPVS 在此做 DNAT（dst 替换为 Pod IP），再经 `POSTROUTING` 转出。
- **conntrack 的作用**：记录 NAT 映射（ClusterIP:port <-> PodIP:port），回程数据包自动触发反向 DNAT，将 src 从 Pod IP 还原为 ClusterIP，PodA 感知不到 PodB 的真实 IP。

### 请求段

```
PodA (10.244.1.10) @ node1 (192.168.1.101)
   |
   | 1. 发包 src=10.244.1.10  dst=ClusterIP(10.96.10.1):80
   v
[node1 内核网络栈]
   |
   | 2. 路由查找 dst=10.96.10.1：命中 kube-ipvs0 本机地址 -> 进 INPUT 链
   |    ★ kube-ipvs0 存在意义：ClusterIP 无真实网卡，kube-proxy 把它以 /32
   |      绑到 dummy 网卡，内核才认为它是"本机"，否则包会被路由出去，IPVS 无法拦截
   |
   | 3. IPVS DNAT（INPUT 链）：dst 10.96.10.1:80 -> PodB(10.244.2.20):80
   |    conntrack 记录映射：(PodA:sport, ClusterIP:80) <-> (PodA:sport, PodB:80)
   |    ★ IPVS 在 INPUT 链做 DNAT（iptables 在 PREROUTING 链，位置不同）
   v
[node1 路由表（Calico BGP 注入）]
   |
   | 4. 查路由：10.244.2.0/24 via node2(192.168.1.102)
   |    ★ 纯 BGP L3 转发，无任何封装，直接 IP 包发出
   v
node2 (192.168.1.102)
   |
   | 5. Calico 本地路由：10.244.2.20 -> PodB 的 veth pair
   v
PodB (10.244.2.20) 收到请求，src=PodA 真实 IP
```

### 响应段

```
PodB (10.244.2.20) @ node2
   |
   | 6. 回包 src=10.244.2.20  dst=10.244.1.10
   v
[node2 路由表（Calico BGP 注入）]
   |
   | 7. 查路由：10.244.1.0/24 via node1(192.168.1.101)，直接转发
   v
[node1 内核网络栈]
   |
   | 8. conntrack 命中 -> 反 DNAT：src 10.244.2.20 -> 10.96.10.1（ClusterIP）
   |    ★ 必须做反 DNAT 的原因：PodA 发起连接时对端是 ClusterIP，
   |      若响应 src 是 PodB 真实 IP，PodA 的 TCP 连接四元组匹配不上，
   |      会直接丢包。conntrack 负责在回程自动完成这个还原。
   v
PodA 收到响应，src=ClusterIP(10.96.10.1)，感知不到 PodB 真实 IP
```

---

## 2. Headless Service（ClusterIP: None）

**本质**：**纯 DNS 服务发现**，kube-proxy 完全不参与，不创建任何 IPVS 规则，不分配 ClusterIP。CoreDNS 对该 Service 名直接返回所有**后端 Pod IP 的 A 记录**（多条）。调用方自己决定用哪个 IP（DNS 客户端负载均衡）。

- **与 ClusterIP 的核心区别**：无 `kube-ipvs0` 绑定，无 IPVS virtual server，数据包完全走 Calico BGP 路由直达 Pod，中间没有任何 NAT。
- **StatefulSet 场景**：每个 Pod 有独立稳定的 DNS 记录 `pod-0.svc.ns.svc.cluster.local`，Pod 重建后 IP 变了但 DNS 名不变。

### 请求段

```
PodA (10.244.1.10) @ node1
   |
   | 1. DNS 查询 headless-svc.default.svc.cluster.local
   v
CoreDNS
   |
   | 2. 直接返回多条 A 记录（Pod 真实 IP）：
   |      10.244.2.20 (PodB)
   |      10.244.3.30 (PodC)
   |    ★ 无 IPVS 规则原因：没有 ClusterIP，kube-proxy 无 VIP 可绑，
   |      kube-ipvs0 无条目，整个 kube-proxy 对此 Service 无感知
   |    ★ 负载均衡责任转移：由 DNS 客户端自行选择用哪个 IP（客户端侧 LB）
   v
PodA 选取一个 Pod IP，直接发包
   |
   | 3. 发包 src=10.244.1.10  dst=10.244.2.20:80（无任何 NAT）
   v
[node1 Calico BGP 路由]
   |
   | 4. 查路由：10.244.2.0/24 via node2，直接 L3 转发，无封装
   v
PodB (10.244.2.20) 收到请求，src=PodA 真实 IP（全程透明，无 NAT）
```

### 响应段

```
PodB (10.244.2.20) @ node2
   |
   | 5. 回包 src=10.244.2.20  dst=10.244.1.10（裸 IP，无任何 NAT）
   v
[node2 Calico BGP 路由]
   |
   | 6. 查路由：10.244.1.0/24 via node1，直接 L3 转发
   v
PodA 收到响应，src=PodB 真实 IP(10.244.2.20)
   ★ 全程无 NAT，连接四元组完全透明，
     适合 gRPC 连接复用等需要感知对端真实身份的场景
```

---

## 3. NodePort

**本质**：**ClusterIP 的超集**。在 ClusterIP 全部 IPVS 规则基础上，额外在**每个节点**的物理网卡 IP:NodePort 上再创建一组 IPVS 虚拟服务器，使外部流量通过任意节点 IP + 高端口（默认 30000-32767）能进入集群。

- **两组 IPVS 规则**：
    - 组 1（同 ClusterIP）：VIP = ClusterIP:port，供集群内部访问
    - 组 2（NodePort 新增）：VIP = nodeIP:NodePort（每个节点都有），供外部访问
- **externalTrafficPolicy=Cluster（默认）**：IPVS 可能把包 DNAT 到其他节点的 Pod，同时做 SNAT 保证回程路径，但**客户端源 IP 丢失**。
- **externalTrafficPolicy=Local**：只 DNAT 到**本节点** Pod，**不做 SNAT**，客户端源 IP 保留；若本节点无对应 Pod，直接拒绝连接。
- **Local 策略的访问盲区**：某个节点上没有部署对应 Pod 时，用该节点 IP:NodePort 访问直接失败。单独使用要求调用方预知哪些节点有 Pod，实际中不可行。
- **最佳实践：与 LB 配合使用**：前置外部 LB，后端为集群所有节点的 NodePort，并开启 healthcheck（kube-proxy 的 10256 端口）。有 Pod 的节点通过检查，无 Pod 的节点返回 503 被自动摘除，既保留真实客户端 IP 又消除访问盲区。

### externalTrafficPolicy=Cluster 请求段

```
外部客户端 (1.2.3.4:5000) -> node1(192.168.1.101):32000
   |
   | ★ 两组 IPVS 规则：
   |   组1 VIP=ClusterIP:port（集群内访问用）
   |   组2 VIP=nodeIP:NodePort（外部访问命中此组）
   |
   | 1. IPVS DNAT：dst node1:32000 -> PodB(10.244.2.20):80
   |    conntrack 记录：(1.2.3.4:5000, node1:32000) <-> (1.2.3.4:5000, PodB:80)
   |
   | 2. SNAT：src 1.2.3.4:5000 -> node1:EPHEMERAL（如 192.168.1.101:49999）
   |    ★ SNAT 必要性：IPVS 可能把包转到 node2 上的 PodB，若不 SNAT，
   |      PodB 响应会通过 BGP 直接回客户端，绕过 node1，
   |      node1 上的 conntrack 反 DNAT 条目永远不被触发，
   |      客户端收到 src=PodB IP 的包（四元组不匹配）-> 连接断裂
   v
[node1 BGP 路由] -> node2
   |
   | 3. PodB 收到：src=node1:EPHEMERAL  dst=PodB:80
   |    ★ 源 IP 已被 SNAT 抹掉，PodB 看不到真实客户端 IP
   v
PodB (10.244.2.20) 处理请求
```

### externalTrafficPolicy=Cluster 响应段

```
PodB (10.244.2.20) @ node2
   |
   | 4. 回包 src=PodB:80  dst=node1:EPHEMERAL(192.168.1.101:49999)
   v
[node2 BGP 路由] -> node1（dst 是 node1，自然回到 node1）
   |
   | 5. node1 conntrack 命中，两步还原：
   |    反 DNAT：src PodB:80      -> node1:32000
   |    反 SNAT：dst node1:49999  -> 1.2.3.4:5000
   v
外部客户端收到：src=node1:32000  dst=1.2.3.4:5000  ✓
```

### externalTrafficPolicy=Local 请求段

```
外部客户端 (1.2.3.4:5000) -> node1:32000
   |
   +--[node1 有本地 PodB]----------------------------------------------+
   |                                                                   |
   | 1. IPVS DNAT：dst node1:32000 -> 本节点 PodB(10.244.1.20):80     |
   |    conntrack 记录映射                                             |
   |    ★ 不做 SNAT：PodB 在本节点，响应必然经过 node1 网络栈，        |
   |      conntrack 可以正确处理反 DNAT，无需强制回程路径              |
   v                                                                   |
PodB (10.244.1.20) @ node1                                            |
   收到：src=1.2.3.4:5000（真实客户端 IP 完整保留）                    |
                                                                       |
   +--[node1 无本地 Pod]-----------------------------------------------+
   |
   | IPVS 无 RS 条目 -> 连接被直接拒绝
   | ★ healthcheck 端口(10256) 返回 503，上游 LB 据此摘除该节点
   | ★ 访问盲区：单独使用时，调用方必须预知哪些节点有 Pod，实际不可行
```

### externalTrafficPolicy=Local 响应段

```
PodB (10.244.1.20) @ node1
   |
   | 2. 回包 src=PodB:80  dst=1.2.3.4:5000
   v
[node1 内核网络栈]
   |
   | 3. conntrack 命中 -> 反 DNAT：src PodB:80 -> node1:32000
   |    无反 SNAT（请求阶段未做 SNAT，无需还原）
   v
外部客户端收到：src=node1:32000  dst=1.2.3.4:5000
   ★ 整个过程 PodB 看到的入站 src 是真实客户端 IP 1.2.3.4
```

---

## 4. LoadBalancer

**本质**：**NodePort 的超集**。K8s 自身不创建任何 LB，只在 Service 对象的 `.status.loadBalancer.ingress` 字段预留位置，等外部控制器（Cloud Controller Manager 或 MetalLB）填写 externalIP 并真正创建 LB。无论哪种实现，流量进入节点后走的都是标准 NodePort IPVS 规则，LoadBalancer 层只负责把流量"引流"到节点门口。

- **云上**：Cloud Controller Manager 调云 API 创建 LB 实例，后端 target group 指向所有节点的 NodePort。
- **裸金属（MetalLB BGP 模式）**：MetalLB speaker 在每个节点运行，通过 BGP 协议将 externalIP/32 路由宣告给物理路由器，路由器学到后把流量送到对应节点，之后走普通 NodePort 逻辑。

### 云上 LoadBalancer 请求段

```
外部客户端 (1.2.3.4)
   |
   | 1. 访问 externalIP（云 LB 地址）
   v
云 LB
   |
   | 2. LB 健康检查（视 externalTrafficPolicy 而定）：
   |    - Cluster：所有节点 NodePort 均可作为后端
   |    - Local  ：healthcheck 端口(10256) 返回 503 的节点被摘除
   |    ★ K8s 本身不创建 LB，只在 .status.loadBalancer.ingress 预留字段，
   |      由 Cloud Controller Manager 调云 API 创建并回填 externalIP
   v
某个 Node:NodePort
   |
   | 3. 进入标准 NodePort IPVS 流程（依 externalTrafficPolicy 策略处理，同第 3 节）
   v
Pod 处理请求
```

### 云上 LoadBalancer 响应段

```
响应路径完全继承 NodePort 对应策略（Cluster 或 Local）的响应段。
云 LB 通常工作在 L4，不介入响应方向，响应直接从 Node 返回客户端
（或经 LB 透传，视具体云厂商实现而定）。
```

### 裸金属 MetalLB BGP 模式 请求段

```
外部客户端 (1.2.3.4)
   |
   | 1. 访问 externalIP (192.168.200.10)
   v
物理路由器
   |
   | 2. BGP 路由表：192.168.200.10/32 已由 MetalLB speaker 宣告
   |    ★ MetalLB speaker 在每个节点运行，通过 BGP 协议将 externalIP/32
   |      路由宣告给物理路由器，路由器学到后把流量送到对应节点
   v
Node:NodePort
   |
   | 3. 进入标准 NodePort IPVS 流程（同第 3 节）
   v
Pod 处理请求
```

### 裸金属 MetalLB BGP 模式 响应段

```
继承 NodePort 对应策略的响应段，与云上路径无本质区别。
```

---

## 5. ExternalName

**本质**：**纯 DNS CNAME**。kube-proxy 完全不参与，不创建 IPVS 规则，不分配 ClusterIP。CoreDNS 对该 Service 名直接返回一条 CNAME 记录指向外部域名。

- **用途**：给集群内 Pod 提供一个稳定的集群内域名来访问外部服务（如 RDS、第三方 API），外部地址变了只需改 Service，Pod 代码无需修改。
- **注意**：无健康检查、无端口映射、无代理；TLS SNI 场景需特别处理（CNAME 目标与证书 CN 需匹配）。

### 请求段

```
PodA (10.244.1.10) @ node1
   |
   | 1. DNS 查询 external-db.default.svc.cluster.local
   v
CoreDNS
   |
   | 2. 返回 CNAME 记录：
   |      external-db.default.svc.cluster.local -> db.example.com
   |    ★ 无 IPVS 规则、无 ClusterIP，kube-proxy 完全不参与
   |    ★ CNAME 解析是递归的：DNS 客户端继续解析 db.example.com
   |      -> 得到外部 IP（如 203.0.113.5），整个过程在 DNS 层完成
   v
PodA 得到外部 IP 203.0.113.5，直接发包
   |
   | 3. 发包 src=10.244.1.10  dst=203.0.113.5:5432
   |    走节点默认路由出集群，完全走外部网络
   v
外部服务 db.example.com (203.0.113.5) 收到请求
```

### 响应段

```
外部服务 (203.0.113.5)
   |
   | 4. 回包 src=203.0.113.5  dst=10.244.1.10
   |    经外部网络 -> 物理路由器 -> node1 -> PodA
   |    ★ 全程无任何 K8s 组件介入，响应直接来自外部服务真实 IP
   v
PodA 收到响应，src=203.0.113.5（外部服务真实 IP）
```

---

## 完整对比表

| 维度 | ClusterIP | Headless | NodePort | LoadBalancer | ExternalName |
|------|-----------|----------|----------|--------------|--------------|
| ClusterIP 分配 | 是 | 否（None） | 是 | 是 | 否 |
| IPVS 规则 | 是（1 组） | 否 | 是（2 组） | 是（2 组，继承 NodePort） | 否 |
| kube-proxy 参与 | 是 | 否 | 是 | 是 | 否 |
| DNS 记录 | A -> ClusterIP | A -> 全部 Pod IP | A -> ClusterIP | A -> ClusterIP | CNAME -> 外部域名 |
| 外部可达 | 否 | 否 | 是（nodeIP:port） | 是（externalIP） | 否（Pod 主动访问外部） |
| 源 IP 保留 | 否（DNAT） | 是 | 可配置（Local 策略） | 可配置（Local 策略） | 是 |
| 典型场景 | 集群内微服务互访 | StatefulSet、gRPC 自定义 LB | 无云环境对外暴露 | 云上 / MetalLB 对外暴露 | 访问外部 DB / API 别名 |
