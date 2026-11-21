
# Mac 安全装机流程（英文界面 / 德国键盘）

> **前提**：如果 ABM 登记还没解除，抹盘和重装都去不掉它，第 2 步会被 "Remote Management" 画面卡住。建议先去店里处理；店里还能直接做 DFU 恢复，连固件一起重写。

## 0. 德国键盘：终端里输入特殊字符
先确认 Terminal → Settings → Profiles → Keyboard 里的 **Use Option as Meta Key 没有勾选**，否则下面带 ⌥ 的组合键打不出字符。

| 字符 | 按键 | 字符 | 按键 |
|---|---|---|---|
| `\|` | ⌥ + 7 | `\` | ⇧ + ⌥ + 7 |
| `{` | ⌥ + 8 | `}` | ⌥ + 9 |
| `'` | ⇧ + #（Ä 右边） | `"` | ⇧ + 2 |
| `<` | 左 Shift 和 Y 之间的键 | `>` | ⇧ + 同一个键 |
| `/` | ⇧ + 7 | `!` | ⇧ + 1 |
| `=` | ⇧ + 0 | `-` | 句号右边的键 |
| `?` | ⇧ + ß | `&` | ⇧ + 6 |
| `:` | ⇧ + . | `~` | ⌥ + N，再按空格 |

打开终端：⌘ + Space 搜 Terminal，或 Applications → Utilities → Terminal

## 1. 彻底抹盘
- 开机时按住电源键，直到出现 "Loading startup options" → **Options** → **Continue**，进入恢复模式
- 菜单栏 **Recovery Assistant → Erase Mac…**，会抹掉所有卷
  - 也可以：**Disk Utility → View → Show All Devices** → 选内置的 Macintosh HD → **Erase**
- Apple 芯片的 SSD 始终是加密的，抹盘会销毁加密密钥，数据无法恢复，不需要多次覆写
- 抹完重启后必须联网 **Activate Mac**，这一步绕不开
- 如果第 2 步选 A1（DFU 恢复），这一步可以省掉，恢复时会自动抹掉全部内容

## 2. 重装系统

### 选项 A：直接装最新版（推荐，省掉从 Tahoe 升级的时间）

**A1. DFU 恢复（最彻底，会连固件一起重写）**
- 在另一台 Mac 上用 Apple Configurator，或者直接在 Apple Store 做
- 按苹果官方步骤让这台 Mac 进入 DFU 模式，用支持数据传输的 USB-C 线连接
- 在 Apple Configurator 里右键这台设备 → **Restore**，会直接装上这台 Mac 能用的最新 macOS

**A2. U 盘安装器（需要另一台 Mac 来制作）**
- 在另一台 Mac 上，先列出可下载的版本，再按列表里的版本号下载完整安装器：
~~~
softwareupdate --list-full-installers
softwareupdate --fetch-full-installer --full-installer-version 27.2.1
~~~
- U 盘至少 32GB，在 **Disk Utility** 里抹掉：Format 选 **Mac OS Extended (Journaled)**，Scheme 选 **GUID Partition Map**，Name 填 **MyVolume**
- 制作安装盘（会清空 U 盘，提示时按 Y 再回车）：
~~~
sudo "/Applications/Install macOS Golden Gate.app/Contents/Resources/createinstallmedia" --volume /Volumes/MyVolume
~~~
- 启动：关机 → 插上 U 盘 → 按住电源键直到 "Loading startup options" → 选安装器 → **Continue**
- 安装过程中仍然需要联网：Apple 芯片要从苹果获取机型专属的固件信息

### 选项 B：恢复模式里重装
- 选 **Reinstall macOS Tahoe**（装的是恢复系统对应的版本，很可能是 Tahoe）
- 到桌面后先在 **System Settings → General → Software Update → Upgrade Now** 升级到最新版，然后再断网
- 走这条路的话，建议先做第 4 步（Lockdown、防火墙）再升级，升完再断网

### Setup Assistant 里的选择（A、B 都一样）
- **Remote Management**：能跳过就跳过，出现就拍照
- **Migration Assistant**：**Not Now**
- **Sign In with Your Apple Account**：**Set Up Later** → **Skip**
- **Create a Computer Account**：用不带真名的名字，它会决定用户文件夹名和默认电脑名
- **Enable Location Services**：取消勾选 → **Continue** → **Don't Use**；**Select Your Time Zone** 里手动选
- **Analytics**：取消勾选 **Share Mac Analytics with Apple** 和 **Share crash and usage data with app developers**
- **Screen Time**：**Set Up Later**
- **Siri**：取消勾选 **Enable Ask Siri**
- **Apple Intelligence**：**Not Now**
- **FileVault Disk Encryption**：勾选 **Turn on FileVault**；让你选恢复方式时不要选 iCloud，把恢复密钥抄在纸上
- **Touch ID**：可以选 **Set Up Touch ID Later**
- 注意：不登录 Apple 账户就没有 Find My 和激活锁，Mac 丢了没法远程定位或锁定

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
- 也可以在 **System Settings → Wi-Fi** 里，点已知网络旁边的 **…** → **Forget This Network…**
- 蓝牙：菜单栏 **Control Center → Bluetooth** 关闭（系统没有自带的命令行工具）
- Mac 的无线模块拔不掉，只能用软件关闭
- 用 `sw_vers` 确认系统版本；以后每次联网，第一件事先更新系统

## 4. 开启 Lockdown Mode 和防火墙
- **Lockdown Mode**：**System Settings → Privacy & Security → Lockdown Mode → Turn On… → Turn On & Restart**
- **防火墙**：图形界面在 **System Settings → Network → Firewall** → 打开 → **Options…** → 勾选 **Block all incoming connections** 和 **Enable stealth mode**（不回应 ping 和端口探测）。也可以用命令设置并检查：
~~~
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setblockall on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
/usr/libexec/ApplicationFirewall/socketfilterfw --getblockall
/usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode
~~~
- 同一步顺手做：
  - **System Settings → General → Sharing**：全部关闭
  - **System Settings → General → AirDrop & Handoff**：**AirDrop** 选 **No One**，关闭 **Allow Handoff between this Mac and your iCloud devices** 和 **AirPlay Receiver**
- 改电脑名（图形界面：**General → About → Name**，以及 **General → Sharing → Local hostname → Edit…**），或者用命令：
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
- 图形界面也看一眼：**System Settings → General → Device Management** 里应该是空的

## 5. 清空 Dock 里的 App
- 图形界面：在 Dock 上右键 App → **Options → Remove from Dock**；再到 **System Settings → Desktop & Dock** 关闭 **Show suggested and recent apps in Dock**
- 或者用命令一次清空：
~~~
defaults write com.apple.dock persistent-apps -array
defaults write com.apple.dock show-recents -bool false
killall Dock
~~~
- 执行后 Dock 只剩 Finder 和 Trash
- 这一步只能防误点：WebKit 是系统组件，Mail、Messages、App Store、Quick Look 都在用；Safari 在系统卷里，删不掉
- 真正的零点击入口（Messages、FaceTime）在没登录 Apple 账户、又断了网的情况下基本用不上；实际起防护作用的是 Lockdown Mode
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
重启（ → **Restart…**）后验证：
~~~
sudo pfctl -s info | head -n 1
sudo pfctl -a user.block -sr
sudo pfctl -s References
~~~
- 应显示 `Status: Enabled` 和 16 条规则；References 里有 pfctl（你的启动项）和 socketfilterfw（系统防火墙）两项
- 这个启动项会出现在 `sfltool dumpbtm` 和 **System Settings → General → Login Items & Extensions** 里，是你自己加的
- 每次系统更新后，检查 `/etc/pf.conf` 里那两行还在不在

## 7. Little Snitch：关掉苹果默认规则，Alert Mode 逐个决定

### 7.1 安装与模式
- 安装 Little Snitch，批准系统扩展和网络扩展：**System Settings → Privacy & Security** 里点 **Allow**，以及 **General → Login Items & Extensions → Network Extensions** 里打开 Little Snitch
- 首次启动时，**macOS Services** 和 **iCloud Services** 都不选；**Apple Apps** 和 **Third Party Apps** 也不要通过 **Rule Groups** 右边的 **+** 添加
- 已经装好的话：在 **Rule Groups** 里取消勾选 **macOS Services** 和 **iCloud Services**
- 模式选 **Alert Mode**：没有规则覆盖的连接会弹窗，你选 Allow 或 Deny 后会自动生成对应的规则
- 出厂保护规则（带锁，不能编辑、不能删除，但可以停用）全部停用，由 7.3 里你自己的规则代替：
  - Any Process：**allow local network**（两条）
  - Any Process：**allow incoming ICMP connections**（两条）：只在 IPv6 关掉之后停用；IPv6 开着就保留
  - configd：**allow local network**、**allow incoming UDP connections (dhcp-client)**
  - mDNSResponder：**allow any outgoing connection**
  - 规则列表里其他带锁的出厂规则也检查一遍
- Little Snitch 自己检查更新时也会弹窗，选 Deny，改用离线安装包手动更新

### 7.2 网络准备：DNS 固定为路由器、固定 IP
- 只用路由器一个 DNS 服务器：
~~~
networksetup -setdnsservers Wi-Fi 192.168.178.1
~~~
- 固定 IP（选在 FRITZ!Box 自动分配范围 .20 到 .200 之外），不用 DHCP 就不需要传入规则：
~~~
networksetup -setmanual Wi-Fi 192.168.178.10 255.255.255.0 192.168.178.1
~~~
- 可选，关闭 IPv6（关掉后才能停用 ICMP 那两条）：
~~~
networksetup -setv6off Wi-Fi
~~~
- 检查：
~~~
networksetup -getinfo Wi-Fi
ipconfig getifaddr en0
netstat -rn -f inet | grep default
~~~
  - getinfo 显示 Manual Configuration、`192.168.178.10`、Router `192.168.178.1`
  - getifaddr 显示 `192.168.178.10`；默认路由经由 `192.168.178.1`
  - 两者对不上时：先运行 `networksetup -setdhcp Wi-Fi`，再把上面的 setmanual 和 setdnsservers 重跑一遍；还不行就重启
- **不要用 `ipconfig set …`**：它是测试命令，会在运行时盖掉网络设置，关开 WiFi 都清不掉
- 连别的 WiFi 要用 DHCP 时：勾选回 configd 的 **allow incoming UDP connections (dhcp-client)**，再运行 `networksetup -setdhcp Wi-Fi`

### 7.3 预先建好的放行规则（规则组 Basis）
> Little Snitch 在你登录之前没法弹窗，这段时间里没有规则覆盖的连接会被直接拒绝。DNS、时间这类开机就要用的连接，所以要提前建好规则。

- 新建规则组：点 **Rule Groups** 标题右边的 **+** → **Local Rule Group…** → Name 填 `Basis`，建好后确认已勾选启用
- 新建规则：点主窗口工具栏的 **(+)**；**Port** 和 **Protocol** 要先点 **More details** 才会显示；Protocol 填协议号：TCP 是 6，UDP 是 17
- 方向：四条都选 **Outgoing**（由 Mac 发起连接，对方的应答会被一起放行）。不要选 **Incoming** 或 **Any Direction**；固定 IP 下不需要任何传入规则

**1. mDNSResponder（DNS）**

| 字段 | 填写 |
|---|---|
| Process | `mDNSResponder` |
| Process owner | System |
| Process identification | with identity check |
| Action | Allow |
| Direction | Outgoing |
| Remote endpoint | IP Addresses → `192.168.178.1` |
| Port | `53` |
| Protocol | `6, 17` |
| Profile or group | Basis |

**2. configd（网络配置）**

| 字段 | 填写 |
|---|---|
| Process | `configd` |
| Process owner | System |
| Process identification | with identity check |
| Action | Allow |
| Direction | Outgoing |
| Remote endpoint | Local Network |
| Port | 不填（任意） |
| Protocol | 不填（任意） |
| Profile or group | Basis |

**3. timed（时间同步）**
- 先在 **System Settings → General → Date & Time** 里打开 **Set time and date automatically**，**Source** 填 `de.pool.ntp.org`

| 字段 | 填写 |
|---|---|
| Process | `timed` |
| Process owner | System |
| Process identification | with identity check |
| Action | Allow |
| Direction | Outgoing |
| Remote endpoint | Domain → `pool.ntp.org` |
| Port | `123` |
| Protocol | `17` |
| Profile or group | Basis |

**4. trustd（证书吊销检查）**

| 字段 | 填写 |
|---|---|
| Process | `trustd` |
| Process owner | System |
| Process identification | with identity check |
| Action | Allow |
| Direction | Outgoing |
| Remote endpoint | Domain → `ocsp.apple.com, ocsp2.apple.com` |
| Port | 不填（任意） |
| Protocol | `6` |
| Profile or group | Basis |

- 弹窗里出现没有齿轮图标的 trustd（用户进程）时，同样只允许这两个域名、方向 Outgoing，规则寿命选 **forever**

  
  
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
