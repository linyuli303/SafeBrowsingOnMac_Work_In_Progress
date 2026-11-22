# Final Cut Pro 离线导出与恢复（同一台 Mac 重装系统后免登录 App Store）

## 1. 确认收据存在

```
ls -l "/Applications/Final Cut Pro.app/Contents/_MASReceipt/receipt"
```

## 2. 导出为 DMG

```
hdiutil create -srcfolder "/Applications/Final Cut Pro.app" -volname "Final Cut Pro" -format UDZO ~/Desktop/FinalCutPro.dmg
shasum -a 256 ~/Desktop/FinalCutPro.dmg
```

记下哈希值，把 DMG 拷到 U 盘或移动硬盘。

### 附加内容（只有下载过才需要）

```
sudo hdiutil create -srcfolder "/Library/Audio/Apple Loops" -format UDZO ~/Desktop/AppleLoops.dmg
sudo hdiutil create -srcfolder "/Library/Application Support/Final Cut Pro" -format UDZO ~/Desktop/FCP-Support.dmg
```

文件夹不存在的话，跳过对应的那条命令。

## 3. 新装系统后恢复

```
# sha：7f2461c25a894f74cdbc029f7022208420677f4ece7cbc8c85c5224aeefca771
shasum -a 256 /Volumes/你的U盘/FinalCutPro.dmg
hdiutil attach /Volumes/你的U盘/FinalCutPro.dmg
sudo ditto "/Volumes/Final Cut Pro/Final Cut Pro.app" "/Applications/Final Cut Pro.app"
sudo chown -R root:wheel "/Applications/Final Cut Pro.app"
hdiutil detach "/Volumes/Final Cut Pro"
```

附加内容的 DMG 用同样的方法恢复，用 `sudo ditto` 拷回原来的路径：

- `/Library/Audio/Apple Loops`
- `/Library/Application Support/Final Cut Pro`

## 4. 验证签名

```
codesign --verify --deep --strict -v "/Applications/Final Cut Pro.app"
spctl -a -vv "/Applications/Final Cut Pro.app"
```

正确结果：显示 `accepted`，来源为 `Mac App Store`。

## 注意事项

- **只适用于同一台 Mac**：收据绑定的是机器的硬件标识，和系统无关。换一台 Mac 就不能用。
- **收据校验失败时**：FCP 打开会要求登录 Apple ID 刷新收据。这时只能登录一次（在"系统设置 → Apple 账户 → 媒体与购买项目"里登录即可）。
- **更新**：只能通过 App Store 或 mas（需要登录）。离线副本会一直停在导出时的版本，更新后要重新导出。
- **Little Snitch**：记得备份规则。FCP 首次启动可能尝试联网，拦掉一般不影响使用。

## 附：用命令行安装（mas）

- App ID：`424389933`
- 安装包：从 https://github.com/mas-cli/mas/releases 下载 `.pkg`。安装包没有签名，要核对 SHA-256：

```
shasum -a 256 ~/Downloads/mas-7.0.0-arm64.pkg
curl -s https://api.github.com/repos/mas-cli/mas/releases/latest | grep -E '"name"|"digest"'
sudo installer -pkg ~/Downloads/mas-7.0.0-arm64.pkg -target /
sudo mas install 424389933
```

- 前提：已经在"系统设置 → 媒体与购买项目"里登录了买 FCP 的那个 Apple ID。
- 网络：防火墙要放行 `mas`，以及系统的 `storedownloadd`、`appstoreagent`、`commerce` 这几个进程访问 `*.apple.com`、`*.mzstatic.com`、`*.cdn-apple.com`。
