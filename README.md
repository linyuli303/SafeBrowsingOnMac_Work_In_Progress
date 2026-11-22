# Mac 安全装机完整记录（2026-11-22）

环境：MacBook Air（Apple Silicon），英文界面，德语键盘；路由器 FRITZ!Box 192.168.178.1；Little Snitch 6.5（7303）

## 原则
- 能离线就离线：联网前先把所有防护配好
- 不用任何苹果服务；不开 Safari、Mail、帮助、App Store；Finder 里不要用空格预览 HTML 文件
- 不接受任何入站连接；出站按白名单放行
- 账户名用中性名字（全名会出现在终端提示符和 Little Snitch 备份里）；对外贴终端输出前先删掉提示符

## 0. 德语键盘
- Terminal → Settings → Profiles → Keyboard：Use Option as Meta Key 保持关闭
- `|` ⌥7　`\` ⇧⌥7　`[` `]` ⌥5 ⌥6　`{` `}` ⌥8 ⌥9　`~` 先按 ⌥N 再按空格
- `'` ⇧#　`"` ⇧2　`/` ⇧7　`=` ⇧0　`?` ⇧ß　`!` ⇧1　`&` ⇧6　`:` ⇧.
- `<` 是 Y 左边那个键，`>` 是 ⇧ 加同一个键

## 1. 抹盘
- 开机时按住电源键 → Options → Recovery Assistant → Erase Mac…

## 2. 重装系统（三选一）
- A1（怀疑本机已被控制时首选）：用另一台可信的 Mac，通过 Apple Configurator 做 DFU Restore，固件和系统一起重刷
- A2：用 U 盘离线安装器装最新的 Golden Gate
~~~
softwareupdate --list-full-installers
softwareupdate --fetch-full-installer --full-installer-version 27.2.1
codesign --verify --deep --strict --verbose=2 "/Applications/Install macOS Golden Gate.app"
codesign -dv --verbose=4 "/Applications/Install macOS Golden Gate.app" 2>&1 | grep Authority
sudo "/Applications/Install macOS Golden Gate.app/Contents/Resources/createinstallmedia" --volume /Volumes/MyVolume
~~~
  - Authority 应依次为 Software Signing、Apple Code Signing Certification Authority、Apple Root CA
  - 把 MyVolume 换成你 U 盘的卷名
- B：Recovery → Reinstall macOS（装的是 Tahoe），装完再用离线安装器升级
- 设置助理：
  - Remote Management：能跳过就跳过（出现这一屏，说明设备还挂在 ABM 里）
  - Apple Account：Set Up Later → Skip
  - 账户名：用中性名字
  - Location Services：关
  - Analytics：全部关
  - Screen Time：Set Up Later
  - Siri：关
  - Apple Intelligence：Not Now
  - FileVault：开；恢复方式选恢复密钥，不用 iCloud；密钥离线保存
- Apple Silicon 激活必须联网，激活完成后立刻执行第 3 步

## 3. 立刻断网（重中之重）
~~~
networksetup -setairportpower en0 off
networksetup -getairportpower en0
sudo networksetup -removepreferredwirelessnetwork en0 "WiFi名称"
~~~
- 全部设置完成之前不联网

## 4. 系统设置（离线）
- Privacy & Security → Lockdown Mode → Turn On…（会重启）
- 防火墙：
~~~
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setblockall on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getblockall
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode
~~~
- General → Sharing：全部关闭
- General → AirDrop & Handoff：AirDrop 设为 No One；Allow Handoff 关；AirPlay Receiver 关
- 时间源（必须在 Little Snitch 规则生效前改）：
~~~
sudo systemsetup -setnetworktimeserver de.pool.ntp.org
cat /etc/ntp.conf
~~~
  - 如果出现 Error:-99 可以忽略，以 /etc/ntp.conf 显示 `server de.pool.ntp.org` 为准
  - Date & Time → Set time and date automatically 保持开启
- 检查设备管理和 FileVault：
~~~
fdesetup status
profiles status -type enrollment
~~~
  - 应显示 FileVault is On；两项 enrollment 都是 No
  - General → Device Management 里应该是空的

## 5. 清空 Dock
~~~
defaults write com.apple.dock persistent-apps -array
defaults write com.apple.dock show-recents -bool false
killall Dock
~~~

## 6. pf
~~~
sudo tee /etc/pf.anchors/user.block > /dev/null <<'EOF'
block out quick on ! lo0 proto tcp to any port { 21 23 135 139 445 3389 }
block out quick on ! lo0 proto udp to any port { 137 138 1900 5353 5355 }
block in quick on ! lo0 proto tcp to any port { 22 445 548 3283 5900 }
EOF
sudo pfctl -vnf /etc/pf.anchors/user.block
echo 'anchor "user.block"' | sudo tee -a /etc/pf.conf
echo 'load anchor "user.block" from "/etc/pf.anchors/user.block"' | sudo tee -a /etc/pf.conf
sudo pfctl -vnf /etc/pf.conf
sudo pfctl -f /etc/pf.conf
~~~
- 检查：
~~~
sudo pfctl -s info | head -n 1
sudo pfctl -a user.block -sr
sudo pfctl -s References
~~~
  - 第一行应显示 Enabled；规则应列出 16 条展开后的规则
- 开机持久化：防火墙开机时会自动加载 pf（已验证重启后仍然生效）
- 出错时重置：
~~~
sudo sed -i '' '/user\.block/d' /etc/pf.conf
tail -n 2 /etc/pf.conf
~~~
  - 最后一行应为 `load anchor "com.apple" from "/etc/pf.anchors/com.apple"`
- 可选（防火墙被关闭时，pf 照样开机加载）：
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

## 7. 网络（离线配置）
~~~
networksetup -setmanual Wi-Fi 192.168.178.10 255.255.255.0 192.168.178.1
networksetup -setdnsservers Wi-Fi 192.168.178.1
networksetup -setv6off Wi-Fi
networksetup -getinfo Wi-Fi
networksetup -getdnsservers Wi-Fi
~~~
- 应显示 Manual Configuration、IP 192.168.178.10、Router 192.168.178.1、IPv6: Off；DNS 只有 192.168.178.1
- FRITZ!Box 上游用 DNS over TLS
- 不要用 `ipconfig set`（它会覆盖设置）；测试不要用 dig 或 ping
- 恢复：
  - 改回 DHCP：`networksetup -setdhcp Wi-Fi`，同时重新勾选 configd 的 dhcp-client 规则
  - 恢复 IPv6：`networksetup -setv6automatic Wi-Fi`
  - 清空 DNS 设置：`networksetup -setdnsservers Wi-Fi empty`

## 8. 安装软件（离线安装包）
- 每个安装包都要验证：
~~~
shasum -a 256 ~/Downloads/文件名.dmg
hdiutil attach ~/Downloads/文件名.dmg -nobrowse -readonly
codesign --verify --deep --strict --verbose=2 "/Volumes/卷名/App名.app"
codesign -dv --verbose=4 "/Volumes/卷名/App名.app" 2>&1 | grep -E "TeamIdentifier|Authority"
~~~
- 各 App 的 Team ID 应该是：
  - Little Snitch：MLZF7K7B5R
  - Zen：9V5K9TP787
  - Brave：KL8N8XSYF4
  - Netiquette：VBG97UB4TA
  - 不装 LuLu
- Brave：
  - Settings → Privacy and security → Security → Use secure DNS：关
  - Site and Shields Settings → V8 optimizer：不允许
  - brave://flags 里的 brave-v8-jitless-mode：Enabled
  - 只用来看 YouTube
- Zen：
  - Settings → Privacy & Security → DNS over HTTPS：Off
  - about:config 里设为 false：network.captive-portal-service.enabled、network.connectivity-service.enabled、app.shield.optoutstudies.enabled
  - 定期更新，Mozilla 的安全数据会随版本一起更新

## 9. Little Snitch

### 9.1 安装与导入
- 安装时批准系统扩展和网络内容过滤
- 联网前在离线状态下导入：File → Restore from Backup…
- 新系统的第一个账户 UID 也是 501，可以直接导入
- 导入前先核对备份：它是纯文本 JSON，重点查有没有意料之外的 allow 规则

### 9.2 设置
- Alert Mode
- DNS Encryption：关
- Security：Allow access via Terminal 关、Allow GUI scripting 关、Allow Global Rule Editing 关

### 9.3 规则组
- macOS Services 关、iCloud Services 关、Basis 开、Apple-Block 开

### 9.4 Basis（8 条，Identification 都是 Code ID，都没有端口和协议限制，另有注明的除外）
| 进程 | Owner | Action | Direction | To |
|---|---|---|---|---|
| mDNSResponder | System | Allow | Outgoing | IP Addresses → 192.168.178.1，Port 53，Protocol any |
| configd | System | Allow | Outgoing | Local Network |
| configd | System | Deny | Any Direction | Any Server |
| timed | System | Allow | Outgoing | Domains → de.pool.ntp.org |
| trustd | System | Allow | Outgoing | Hostnames → certs.apple.com, crl.apple.com, ocsp.apple.com, ocsp2.apple.com |
| trustd | Me | Allow | Outgoing | Hostnames → 同上 |
| nsurlsessiond | System | Allow | Outgoing | Hostnames → valid.apple.com |
| softwareupdated | System | Allow | Outgoing | Domains → apple.com |

### 9.5 Apple-Block（2 条）
- Process：Any Process；Owner：System 一条、Me 一条
- Action：Deny；Direction：Outgoing；To：Domains → apple.com, icloud.com
- 没有端口和协议限制；选 Deny 后编辑器里不再显示 Require

### 9.6 出厂规则状态
- 已关闭：
  - configd 的 incoming UDP 68（dhcp-client）
  - Any Process 的 incoming ICMP（System 和 Me）
  - mDNSResponder 的 allow any outgoing
  - ocspd 的 allow any
  - Any Process 的 Local Network（System 和 Me）
- 保持开启：
  - trustd 的 allow any（System 和 Me），负责非苹果的证书服务器
  - airportd、bootpd、configd、mDNSResponderHelper 的 BPF 规则
  - loopback 隐藏规则

### 9.7 单进程拒绝规则
- System：
  - mDNSResponder：拒绝所有入站；拒绝出站到 224.0.0.251；拒绝出站到 icloud.com
  - 拒绝 Any：apsd、cloudd、ctcategories、geod、managedappdistributiond、netbiosd、mobileactivationd、mobileassetd、nsurlsessiond、syspolicyd、wifivelocityd、Little Snitch networking
  - 拒绝到 192.168.178.1：airportd、symptomsd
- Me：
  - 拒绝 Any：appleaccountd、cloudd、geod、helpd、icloudmailagent、itunescloudd、parsec-fbf、promotedcontentd、tipsd、Weather、WeatherWidget、webprivacyd、Safari SafeBrowsing、Mail、networkserviceproxy、ocspd
  - 拒绝 apple.com：AMPLibraryAgent、AuthenticationServicesAgent、amsengagementd、amsondevicestoraged、adprivacyd、appstoreagent、identityservicesd、managedappdistributionagent、nsurlsessiond、swtransparencyd
  - 拒绝 icloud.com：CloudTelemetryService、FeatureAccessAgent、familycircled、iCloudNotificationAgent、nsurlsessiond
  - 其它：swtransparencyd 拒绝 cloudflare.com；sharingd 拒绝 .local 设备的 8770 端口
- 拒绝 apple.com 和 icloud.com 的单进程规则已被 Apple-Block 覆盖，保留作后备，临时关掉 Apple-Block 时仍然生效

### 9.8 浏览器和其它 App（Owner 都是 Me）
- Brave：
  - Allow，Outgoing，TCP 443 → Domains：ggpht.com, googlevideo.com, gstatic.com, youtube-nocookie.com, youtube.com, ytimg.com
  - Allow，Outgoing，UDP 443 → Domains：ggpht.com, googlevideo.com, youtube.com, ytimg.com
  - Allow，Outgoing → Hostnames：safebrowsing.brave.com
  - Deny，Any Direction → Any Server
- Zen（白名单模式）：
  - Deny，Outgoing → Any Server
  - Allow，Outgoing → Domains：
    - claude.ai, claude.com
    - challenges.cloudflare.com, chatgpt.com, featureassets.org, featuregates.org, oaistatic.com, oaiusercontent.com, openai.com, openaimerge.com, prodregistryv2.org, statsig.com, statsigapi.net, workos.com, workoscdn.com
    - github.com, githubassets.com, githubcopilot.com, githubusercontent.com
    - duckduckgo.com, google.com, wikipedia.org
    - captcha-delivery.com；hcaptcha.com；octocaptcha.com
    - objective-see.com, objective-see.org
    - getutm.app；virustotal.com；zen-browser.app（浏览器更新）
  - 不放行 Mozilla 的 Remote Settings，靠更新 Zen 来刷新
- Netiquette：Allow，Outgoing → Domains：objective-see.org
- Terminal via curl：Deny，Outgoing → Domains：githubusercontent.com（需要从 GitHub 下载时临时取消勾选）
- Little Snitch Software Update：Deny → Domains：obdev.at（改为手动更新）

### 9.9 规则优先级与注意事项
- 优先级：Hostnames > Domains > … > Any；同为 Domains 时，列表短的、标签少的优先，所以 apple.com 之下的例外一律用 Hostnames；具体进程胜过 Any Process
- 弹窗里点 Deny 时，范围不要选 Any Connection
- Until Quit 指的是该 App 退出时失效；临时规则到期后会变成 suggestion
- 登录前无法弹窗，所以必需的放行规则要提前建好
- 备份文件没有加密，里面有账户全名

### 9.10 导出最终备份前要清理
- 所有 suggestion
- mDNSResponder 的"deny incoming 192.168.178.155"
- 可选：只针对 IPv6 的两条规则（mDNSResponder → ff02::fb，symptomsd → fe80::…）

## 10. 已知取舍
- XProtect：cloudd 被拒，只能通过 Software Update 更新；必要时运行 `sudo xprotect update`，用 `xprotect version` 查看版本
- Gatekeeper 的在线公证检查被挡（syspolicyd），只能靠 App 自带的公证票据，加上自己用 codesign 校验
- Little Snitch 不会自动更新，要手动下载离线安装包更新
- Zen 的证书吊销列表等安全数据只能随版本更新

## 11. 装完验证
~~~
ipconfig getifaddr en0
scutil --dns | grep nameserver
ifconfig en0 | grep inet6
ifconfig en0 | grep ether
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
dscacheutil -q host -a name debian.org
curl -sS -o /dev/null https://www.apple.com
softwareupdate -l
cat /etc/ntp.conf
systemextensionsctl list
sudo pfctl -s info | head -n 1
sudo lsof -nP -a -i -c at.obdev
~~~
- IP 是 192.168.178.10
- DNS 服务器只有 192.168.178.1（显示两行是正常的）
- inet6 没有输出
- MAC 地址第一组的第二个字符是 2、6、A、E 之一（说明在用私有地址）
- debian.org 能返回 IP，Basis 里 mDNSResponder 那条规则的 Used 在增加
- curl 连 apple.com 应该失败
- softwareupdate -l 能正常运行完
- ntp.conf 显示 de.pool.ntp.org；把 Set time and date automatically 关掉再打开后，timed 规则的 Used 会增加
- 系统扩展只有 Little Snitch
- pf 显示 Enabled
- Little Snitch 进程没有任何网络连接，也就没有连 Quad9

## 12. 下次装机顺序
抹盘 → 装系统 → 设置助理（激活）→ 立刻断网 → 系统设置（Lockdown、防火墙、共享、时间源）→ 清空 Dock → pf → 网络（固定 IP、DNS、关 IPv6）→ 离线安装 Little Snitch 并导入备份 → 安装 Zen、Brave、Netiquette 并配置 → 连 Wi-Fi，私有地址设为 Rotating → 按第 11 节验证

## 待办
- 下周去店里处理 ABM 设备管理的解除
- 定期手动更新 Little Snitch 和 Zen
