# Mac OS

# Mac 安全装机流程

> **前提**：如果 ABM 登记还没解除，抹盘和重装都去不掉它，第 2 步会被"远程管理"画面卡住。建议先去店里处理；店里还能直接做 DFU 恢复，连固件一起重写。

## 1. 彻底抹盘
- 开机时按住电源键 → 选项 → 继续，进入恢复模式
- 菜单栏选"恢复助理 → 抹掉Mac"，会抹掉所有卷
  - 也可以用磁盘工具：显示 → 显示所有设备 → 选内置的 Macintosh HD → 抹掉
- Apple 芯片的 SSD 始终是加密的，抹盘会销毁加密密钥，数据无法恢复，不需要多次覆写
- 抹完重启后必须联网"激活Mac"，这一步绕不开
- 如果第 2 步选 A1（DFU 恢复），这一步可以省掉，恢复时会自动抹掉全部内容

## 2. 重装系统

### 选项 A：直接装最新版（推荐，省掉从 Tahoe 升级的时间）

**A1. DFU 恢复（最彻底，会连固件一起重写）**
- 在另一台 Mac 上用 Apple Configurator，或者直接在 Apple Store 做
- 按苹果官方步骤让这台 Mac 进入 DFU 模式，用支持数据传输的 USB-C 线连接
- 在 Apple Configurator 里右键这台设备 → 恢复（Restore），会直接装上这台 Mac 能用的最新 macOS

**A2. U 盘安装器（需要另一台 Mac 来制作）**
- 在另一台 Mac 上，先列出可下载的版本，再按列表里的版本号下载完整安装器：
~~~
softwareupdate --list-full-installers
softwareupdate --fetch-full-installer --full-installer-version 27.2.1
~~~
- U 盘至少 32GB，在磁盘工具里抹成"Mac OS扩展（日志式）"，命名为 MyVolume，然后制作安装盘（会清空 U 盘，提示时输入 Y）：
~~~
sudo "/Applications/Install macOS Golden Gate.app/Contents/Resources/createinstallmedia" --volume /Volumes/MyVolume
~~~
- 启动：关机 → 插上 U 盘 → 按住电源键，直到出现"正在载入启动选项" → 选安装器 → 继续
- 安装过程中仍然需要联网：Apple 芯片要从苹果获取机型专属的固件信息

### 选项 B：恢复模式里"重新安装macOS"
- 装的是恢复系统对应的版本，很可能是 Tahoe（macOS 26）
- 到桌面后要先在"系统设置 → 通用 → 软件更新"里升级到最新版，然后再断网
- 走这条路的话，建议先做第 4 步（Lockdown、防火墙）再升级，升完再断网

### 设置助理里的选择（A、B 都一样）
- **远程管理**：能跳过就跳过，出现就拍照
- **迁移助理**：以后
- **Apple 账户**：稍后设置 → 跳过
- **电脑账户**：用不带真名的名字，它会决定用户文件夹名和默认电脑名
- **定位服务**：关闭；时区手动选
- **分析**：全部取消勾选
- **屏幕使用时间、Siri、Apple 智能**：全部跳过
- **文件保险箱（FileVault）**：开启；让你选恢复方式时不要选 iCloud，把恢复密钥抄在纸上
- 注意：不登录 Apple 账户就没有"查找"和激活锁，Mac 丢了没法远程定位或锁定

## 3. 进桌面后立刻断网（重中之重）
先查 WiFi 的设备名，一般是 `en0`；如果不是，把下面命令里的 en0 换掉：
~~~
networksetup -listallhardwareports
~~~
关闭 WiFi 并确认：
~~~
networksetup -setairportpower en0 off
networksetup -getairportpower en0
~~~
删掉装机时连过的 WiFi，防止以后自动重连（把引号里换成你的网络名）：
~~~
sudo networksetup -removepreferredwirelessnetwork en0 "WiFi名称"
~~~
- 蓝牙在控制中心里关闭，系统没有自带的命令行工具
- Mac 的无线模块拔不掉，只能用软件关闭
- 用 `sw_vers` 确认系统版本；以后每次联网，第一件事先更新系统

## 4. 开启 Lockdown Mode 和防火墙
- **Lockdown Mode**：系统设置 → 隐私与安全性 → 锁定模式 → 打开，会重启
- **防火墙**：开启、阻止所有传入连接、开启隐身模式（不回应 ping 和端口探测），然后检查状态：
~~~
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setblockall on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
/usr/libexec/ApplicationFirewall/socketfilterfw --getblockall
/usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode
~~~
- 同一步顺手做："共享"里全部关闭；隔空投送设为"不接收"；关闭接力和隔空播放接收器
- 改电脑名，换成不带名字的通用名称：
~~~
sudo scutil --set ComputerName "MacBook"
sudo scutil --set LocalHostName "MacBook"
sudo scutil --set HostName "MacBook"
~~~
- 检查文件保险箱和设备管理（断网也能用）。应显示 `FileVault is On`，后一条的两项都是 No：
~~~
fdesetup status
profiles status -type enrollment
~~~

## 5. 清空程序坞里的 App
~~~
defaults write com.apple.dock persistent-apps -array
defaults write com.apple.dock show-recents -bool false
killall Dock
~~~
- 执行后程序坞只剩访达和废纸篓，也不再显示最近用过的 App
- 这一步只能防误点：WebKit 是系统组件，邮件、信息、App Store、快速查看都在用；Safari 在系统卷里，删不掉
- 真正的零点击入口（信息、FaceTime）在没登录 Apple 账户、又断了网的情况下基本用不上；实际起防护作用的是 Lockdown Mode
- 断网之后，这一步放在第 4 步之前或之后都可以

## 6. pf 端口拦截 + 开机自启
写规则文件，并检查语法：
~~~
sudo tee /etc/pf.anchors/user.block > /dev/null <<'EOF'
block out quick on ! lo0 proto tcp to any port { 21 23 135 139 445 3389 }
block out quick on ! lo0 proto udp to any port { 137 138 1900 5353 5355 }
block in quick on ! lo0 proto tcp to any port { 22 445 548 3283 5900 }
EOF
sudo pfctl -vnf /etc/pf.anchors/user.block
~~~
挂进系统配置并加载：
~~~
echo 'anchor "user.block"' | sudo tee -a /etc/pf.conf
echo 'load anchor "user.block" from "/etc/pf.anchors/user.block"' | sudo tee -a /etc/pf.conf
sudo pfctl -vnf /etc/pf.conf
sudo pfctl -f /etc/pf.conf
~~~
自己写开机启动项（不依赖系统防火墙）：
~~~
sudo tee /Library/LaunchDaemons/user.pf.plist > /dev/null <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>user.pf</string>
  <key>ProgramArguments</key>
  <array>
    <string>/sbin/pfctl</string>
    <string>-E</string>
    <string>-f</string>
    <string>/etc/pf.conf</string>
  </array>
  <key>RunAtLoad</key><true/>
</dict>
</plist>
EOF
sudo chown root:wheel /Library/LaunchDaemons/user.pf.plist
sudo chmod 644 /Library/LaunchDaemons/user.pf.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/user.pf.plist
~~~
重启后验证：
~~~
sudo pfctl -s info | head -n 1
sudo pfctl -a user.block -sr
sudo pfctl -s References
~~~
- 应显示 `Status: Enabled` 和 16 条规则；References 里有 pfctl（你的启动项）和 socketfilterfw（系统防火墙）两项
- 这个启动项会出现在 `sfltool dumpbtm` 和"登录项与扩展"里，是你自己加的
- 每次系统更新后，检查 `/etc/pf.conf` 里那两行还在不在

## 7. 离线安装浏览器和工具
- 在另一台电脑上从官网下载：Brave、Zen、Little Snitch（obdev.at）、Netiquette（objective-see.org），用 U 盘拷过来
- 官网或 GitHub 发布页给了校验值的话，先核对（换成实际路径）：
~~~
shasum -a 256 /Volumes/U盘名/文件名.dmg
~~~
- 打开 dmg，把 App 拖进"应用程序"，然后逐个验证签名（App 可以从访达直接拖进终端，路径会自动填好）：
~~~
codesign --verify --deep --strict --verbose=2 "/Applications/Brave Browser.app"
codesign -dv --verbose=4 "/Applications/Brave Browser.app" 2>&1 | grep Authority
~~~
  - 第一条应显示 `valid on disk` 和 `satisfies its Designated Requirement`
  - 第二条列出的第一条 Authority 应该是开发者名字：Brave 是 Brave Software, Inc.，Little Snitch 是 Objective Development Software GmbH，Netiquette 是 Objective-See, LLC
- 从 U 盘拷进来的文件不带"隔离"标记，Gatekeeper 第一次打开时不会检查，所以签名必须自己验

### Little Snitch：Brave 只放行 YouTube
- 安装后，在系统设置里批准它的系统扩展和网络扩展
- 建两条规则：
  1. Brave Browser：拒绝连接任何服务器
  2. Brave Browser：允许连接 youtube.com、googlevideo.com（视频流）、ytimg.com（缩略图和播放器文件）、ggpht.com（头像）
- 指定了域名的规则优先级高于"任何服务器"，所以第 2 条会覆盖第 1 条
- 页面或视频加载不全时，在 Little Snitch 的网络监视器里看被拦了哪些域名，常见的还有 gstatic.com、googleapis.com，按需放行
- Brave 设置里的"使用安全DNS"如果开着，要么放行它用的 DNS 服务器，要么关掉，否则域名解析也会被一起挡掉
- Brave 自己的更新和广告过滤规则也要联网：要么额外放行 Brave 的服务器（具体域名在网络监视器里能看到），要么定期用新的离线安装包手动更新
- Netiquette 只是查看当前网络连接的工具，不用设置规则

  
# Brave 设置汇总（2026-11-21）

## 隐私与数据
- **退出时删除数据**：brave://settings/clearBrowserData → Delete data on exit，已勾选 Cookies 和缓存
- **启动设置**：不要选 "Continue where you left off"，否则会话会被恢复，退出时也删不干净
- brave://settings/cookies 里已经没有"关闭时删除"的开关，用上面那项代替
- **Mac 上要用 Cmd+Q 才算真正退出**，点红色按钮只是关窗口。Mac 版没有"后台运行"开关

## 强制私密模式
- 开启（已生效）：
  `defaults write com.brave.Browser IncognitoModeAvailability -integer 2`
- 恢复普通模式：
  `defaults delete com.brave.Browser IncognitoModeAvailability`
- 私密窗口里扩展要单独打开 "Allow in Private"
- 私密模式下容器（Containers）大概率不能用

## 容器
- 开启：brave://settings/braveContent → Enable Containers
- 使用：右键标签页 → Open in Container

## Shields
- 过滤列表：brave://settings/shields/filters
- 被拦掉的 `log_event`、`qoe` 是 YouTube 遥测请求，不影响播放
- 过滤列表和组件可能因为联网限制没法自动更新，可以到 brave://components 查看更新时间

## 待做
- **关闭 JavaScript 优化**：brave://settings/content/v8 → 不允许网站使用 V8 优化器
  （Brave 是 Chromium 内核，macOS 的锁定模式保护不到它）

## Little Snitch 规则（Brave Browser）
- 允许 safebrowsing.brave.com
- 允许 ggpht.com、googlevideo.com、youtube.com、ytimg.com，**UDP 443（QUIC）**
- 允许 ggpht.com、googlevideo.com、gstatic.com、youtube.com，再加上截图里被截断的其余 YouTube 域名（TCP）
  - **google.com 和 googleapis.com 已去掉**，YouTube 照常使用，速度不受影响
  - 去掉的原因：这两个域名下的 Google 表单、云存储、Apps Script 不需要你登录也能被用来往外传数据（你的 IP、打开文件的记录、页面数据）
- 拒绝所有其他出站连接（两条 deny 规则）
- 以后如果出现"请确认你不是机器人"的提示或者播放失败，就到 Little Snitch 里查看 Brave 被拒绝的连接，只补需要的那个子域名，不要把整个大域名加回来

## QUIC / YouTube
- brave://flags/#enable-quic 设为 Default 或 Enabled
- 检查 brave://policy，确认没有 QuicAllowed=false
- `defaults read com.brave.Browser QuicAllowed` 返回 does not exist，说明没有限制
- 加了规则还是 http/1.1：先 Cmd+Q 退出，再删除
  `~/Library/Application Support/BraveSoftware/Brave-Browser/Default/Network Persistent State`
- 验证：DevTools → Network → 右键表头勾选 Protocol → 过滤 `videoplayback`，显示 h3 就是走了 QUIC
- 截图前要把 videoplayback 链接里的 `ip=` 参数（你的公网 IP）打码

## 本地 .html 文件
- 默认用 **Brave** 打开：选中 .html 文件 → Cmd+I → 打开方式选 Brave → 全部更改；.htm 也要单独改一次
- 要在设置默认浏览器之后再改，因为设置默认浏览器时可能会把这个关联覆盖掉
- 好处：Brave 只能连到 YouTube 和 Brave 安全浏览，本地网页里的追踪和数据外传基本连不出去
- 局限：断网挡不住直接写在文件里的漏洞利用，所以陌生的 .html 文件还是尽量别打开，V8 优化也要关掉

## 更新（手动）
- Brave 的更新服务器没有放行，所以要**定期手动更新**，Chromium 的漏洞修补得很频繁
- 从官网下载 .dmg（可以先存到 U 盘），装好后检查签名：
  `codesign -dv --verify --strict "/Applications/Brave Browser.app"`
  `spctl -a -vv "/Applications/Brave Browser.app"`
  输出里应该有 `source=Notarized Developer ID`，`origin=` 那一项应该是 **Brave Software, Inc.**
- 每次更新后都要检查 Little Snitch 的规则还能不能匹配，因为 Code ID 可能会变
- 在 brave://settings/help 可以看当前版本号
