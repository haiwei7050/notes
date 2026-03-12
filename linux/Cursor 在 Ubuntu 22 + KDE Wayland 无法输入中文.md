# Cursor 在 Ubuntu 22 + KDE Wayland 无法输入中文

## 结论（根本原因）

在本机环境（Ubuntu 22 + KDE Plasma 5.24 + Wayland + fcitx5）下，问题根因不是单点，而是 3 个条件叠加：

1. `~/.config/kwinrc` 缺少 `Wayland/InputMethod`，KWin 不会按 Wayland 输入法链路接管 fcitx5。
2. `GTK_IM_MODULE` / `QT_IM_MODULE` 被强制设置，干扰 Electron Ozone 在 Wayland 下的 text-input 路径。
3. 用户级 `~/.config/autostart/fcitx5.desktop` 与 KWin 启动 fcitx5 的机制竞争，导致实例管理不稳定。

修复后确认：`fcitx5-diagnose` 已出现 `Wayland Input method frontend`，且 Cursor 内可正常候选与上屏。

## 版本口径说明

- `qdbus org.kde.KWin /KWin org.kde.KWin.VirtualKeyboard.available` 在本机 Plasma 5.24 返回 `UnknownInterface`，该接口未暴露。
- 因此本场景不再把该 DBus 接口作为唯一验收标准，改为以“Wayland frontend 已连接 + Cursor 实测可输入中文”为最终判定。

## 从 0 到 1 修复命令（省略探测命令）

按顺序执行以下命令：

```bash
# 0) 备份
mkdir -p ~/fcitx5-fix-backup
cp ~/.config/autostart/fcitx5.desktop ~/fcitx5-fix-backup/autostart-fcitx5-custom.desktop
cp ~/.pam_environment ~/fcitx5-fix-backup/pam_environment.before
sudo cp /etc/environment /etc/environment.bak.cursor-ime-$(date +%Y%m%d%H%M%S)

# 1) 配置 KWin 使用 fcitx5（Wayland InputMethod）
kwriteconfig5 --file kwinrc --group Wayland --key InputMethod /usr/share/applications/org.fcitx.Fcitx5.desktop

# 2) 去掉用户级 autostart 竞争实例
rm -f ~/.config/autostart/fcitx5.desktop

# 3) 调整用户会话环境变量（保留 XMODIFIERS）
sed -i 's/^GTK_IM_MODULE /#GTK_IM_MODULE /' ~/.pam_environment
sed -i 's/^QT_IM_MODULE /#QT_IM_MODULE /' ~/.pam_environment
sed -i 's/^XMODIFIERS DEFAULT=@im=fcitx5$/XMODIFIERS DEFAULT=@im=fcitx/' ~/.pam_environment

# 4) 调整系统环境变量（保留 XMODIFIERS）
sudo sed -i 's/^GTK_IM_MODULE=/#GTK_IM_MODULE=/' /etc/environment
sudo sed -i 's/^QT_IM_MODULE=/#QT_IM_MODULE=/' /etc/environment

# 5) 注销并重新登录 Plasma (Wayland)（人工步骤）

# 6) 登录后启动 Cursor（即使有 warning 也不影响参数透传）
cursor --enable-wayland-ime
```

## 重新生成 Cursor 桌面快捷方式（用户级覆盖）

```bash
mkdir -p ~/.local/share/applications
cp /usr/share/applications/cursor.desktop ~/.local/share/applications/cursor.desktop
sed -i 's|^Exec=/usr/share/cursor/cursor %F$|Exec=/usr/share/cursor/cursor --enable-wayland-ime %F|' ~/.local/share/applications/cursor.desktop
sed -i 's|^Exec=/usr/share/cursor/cursor --new-window %F$|Exec=/usr/share/cursor/cursor --enable-wayland-ime --new-window %F|' ~/.local/share/applications/cursor.desktop
update-desktop-database ~/.local/share/applications
```

## 回滚命令（如后续出现回归）

```bash
# 1) 恢复用户级 autostart
cp ~/fcitx5-fix-backup/autostart-fcitx5-custom.desktop ~/.config/autostart/fcitx5.desktop

# 2) 删除 KWin InputMethod
kwriteconfig5 --file kwinrc --group Wayland --key InputMethod --delete

# 3) 恢复用户环境变量
cp ~/fcitx5-fix-backup/pam_environment.before ~/.pam_environment

# 4) 恢复系统环境变量（替换为你实际备份文件名）
sudo cp /etc/environment.bak.cursor-ime-YYYYMMDDHHMMSS /etc/environment

# 5) 回退桌面快捷方式覆盖
rm -f ~/.local/share/applications/cursor.desktop
update-desktop-database ~/.local/share/applications
```
