# Mac 防火墙与无线加固设置

整理日期：2026-11-24 ｜ 设备：MacBook Air（M5）｜ 系统：macOS 26 ｜ 机器名：pc
网络：Wi-Fi 静态 IP 192.168.178.10 / 网关 192.168.178.1（FRITZ!Box）/ IPv6 关闭

## 使用前须知

- 已在 `~/.zshrc` 加入 `setopt interactivecomments`，zsh 可以识别 `#` 注释
- 带 sudo 的长脚本都包在 `sudo -v && { … }` 里，整段粘贴、回车，输一次密码即可
- 防护共四层：pf（内核包过滤）→ 应用防火墙 → Little Snitch（出站按程序控制）→ 关闭无线与广播服务

---

## 一、pf 防火墙

### 规则文件 `/etc/pf.anchors/user.block`

- 出站封禁：FTP 21、Telnet 23、RPC 135、NetBIOS 139、SMB 445、RDP 3389；UDP 137/138、SSDP 1900、mDNS 5353、LLMNR 5355
- 出站其余放行并建立状态，所以只有自己发起的连接的回包能进来
- 入站只放行 DHCP 回复，其余全部丢弃

```
sudo tee /etc/pf.anchors/user.block > /dev/null <<'EOF'
block out quick on ! lo0 proto tcp to any port { 21 23 135 139 445 3389 }
block out quick on ! lo0 proto udp to any port { 137 138 1900 5353 5355 }
pass out quick on ! lo0 all keep state
pass in quick on ! lo0 proto udp from any port 67 to any port 68
block in quick on ! lo0 all
EOF
```

### 主配置 `/etc/pf.user.conf`

不修改系统的 `/etc/pf.conf`，系统更新不会覆盖。

```
sudo tee /etc/pf.user.conf > /dev/null <<'EOF'
scrub-anchor "com.apple/*"
nat-anchor "com.apple/*"
rdr-anchor "com.apple/*"
dummynet-anchor "com.apple/*"
anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
anchor "user.block"
load anchor "user.block" from "/etc/pf.anchors/user.block"
EOF
sudo pfctl -vnf /etc/pf.user.conf
sudo pfctl -f /etc/pf.user.conf
```

### 开机自动加载 `/Library/LaunchDaemons/user.pf.plist`

```
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
    <string>/etc/pf.user.conf</string>
  </array>
  <key>RunAtLoad</key><true/>
</dict>
</plist>
EOF
sudo chown root:wheel /Library/LaunchDaemons/user.pf.plist
sudo chmod 644 /Library/LaunchDaemons/user.pf.plist
plutil -lint /Library/LaunchDaemons/user.pf.plist
sudo launchctl bootout system /Library/LaunchDaemons/user.pf.plist 2>/dev/null
sudo launchctl bootstrap system /Library/LaunchDaemons/user.pf.plist
```

### 紧急回滚

```
sudo pfctl -f /etc/pf.conf
```

---

## 二、macOS 应用防火墙

```
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setblockall on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on
```

---

## 三、关闭无线与广播（保留 Wi-Fi）

```
sudo -v && {
osascript -l JavaScript -e 'ObjC.import("IOBluetooth"); ObjC.bindFunction("IOBluetoothPreferenceSetControllerPowerState", ["void", ["int"]]); $.IOBluetoothPreferenceSetControllerPowerState(0)'
networksetup -listallnetworkservices | tail -n +2 | sed 's/^\*//' | grep -vxE 'Wi-Fi|WLAN' | while read -r s; do sudo networksetup -setnetworkserviceenabled "$s" off; done
for i in awdl0 llw0 ap1; do sudo ifconfig "$i" down 2>/dev/null; done
sudo tee /Library/LaunchDaemons/local.p2p-off.plist > /dev/null <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>local.p2p-off</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/sh</string>
    <string>-c</string>
    <string>for i in awdl0 llw0 ap1; do /sbin/ifconfig $i down 2>/dev/null; done</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>StartInterval</key>
  <integer>10</integer>
</dict>
</plist>
EOF
sudo chown root:wheel /Library/LaunchDaemons/local.p2p-off.plist
sudo chmod 644 /Library/LaunchDaemons/local.p2p-off.plist
sudo launchctl bootout system/local.p2p-off 2>/dev/null
plutil -lint /Library/LaunchDaemons/local.p2p-off.plist && sudo launchctl bootstrap system /Library/LaunchDaemons/local.p2p-off.plist
defaults write com.apple.sharingd DiscoverableMode -string "Off"
defaults write com.apple.NetworkBrowser DisableAirDrop -bool YES
killall sharingd 2>/dev/null
sudo defaults write /Library/Preferences/com.apple.mDNSResponder.plist NoMulticastAdvertisements -bool YES
sudo killall mDNSResponder
defaults -currentHost write com.apple.coreservices.useractivityd ActivityAdvertisingAllowed -bool no
defaults -currentHost write com.apple.coreservices.useractivityd ActivityReceivingAllowed -bool no
defaults -currentHost write com.apple.controlcenter AirplayRecieverEnabled -bool false
sudo launchctl disable system/com.openssh.sshd
sudo launchctl disable system/com.apple.AEServer
sudo launchctl disable system/com.apple.screensharing
sudo launchctl disable system/com.apple.smbd
sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -deactivate -stop
sudo cupsctl --no-share-printers --no-remote-any --no-remote-admin
sudo launchctl disable system/org.cups.cupsd
grep -q port445 /etc/nsmb.conf 2>/dev/null || printf '[default]\nprotocol_vers_map=6\nport445=no_netbios\n' | sudo tee -a /etc/nsmb.conf > /dev/null
sudo launchctl disable system/com.apple.netbiosd
sudo launchctl bootout system/com.apple.netbiosd 2>/dev/null
sudo scutil --set ComputerName "pc"
sudo scutil --set LocalHostName "pc"
sudo scutil --set HostName "pc"
sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.captive.control Active -bool false
sudo pmset -a womp 0
sudo pmset -a powernap 0
sudo pmset -a tcpkeepalive 0
launchctl bootout gui/$(id -u)/local.bluetooth-off 2>/dev/null
mkdir -p ~/Library/LaunchAgents
tee ~/Library/LaunchAgents/local.bluetooth-off.plist > /dev/null <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>local.bluetooth-off</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/osascript</string>
    <string>-l</string>
    <string>JavaScript</string>
    <string>-e</string>
    <string>ObjC.import("IOBluetooth");
ObjC.bindFunction("IOBluetoothPreferenceGetControllerPowerState", ["int", []]);
ObjC.bindFunction("IOBluetoothPreferenceSetControllerPowerState", ["void", ["int"]]);
while (true) {
  if ($.IOBluetoothPreferenceGetControllerPowerState() > 0) {
    $.IOBluetoothPreferenceSetControllerPowerState(0);
  }
  delay(0.2);
}</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>ThrottleInterval</key>
  <integer>1</integer>
</dict>
</plist>
EOF
plutil -lint ~/Library/LaunchAgents/local.bluetooth-off.plist && launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.bluetooth-off.plist
echo "完成"
}
```

蓝牙全局描述文件：`~/Desktop/disable-bluetooth.mobileconfig`，在 系统设置 → 通用 → 设备管理 中安装。

### 手动设置

- 系统设置 → 隐私与安全性 → 定位服务 → 关闭
- Wi-Fi → 当前网络的"详细信息" → 私有 Wi-Fi 地址 → 轮换
- Wi-Fi → 询问是否加入网络 → 关闭；询问是否加入热点 → 永不
- 系统设置 → 通用 → 共享 → 全部关闭

---

## 四、检查清单（每次重启、每次系统更新后执行）

```
sudo pfctl -s info | head -n 1
sudo pfctl -sr | grep user.block
sudo pfctl -a user.block -sr
/usr/libexec/ApplicationFirewall/socketfilterfw --getblockall
/usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode
networksetup -getairportpower en0
osascript -l JavaScript -e 'ObjC.import("IOBluetooth"); ObjC.bindFunction("IOBluetoothPreferenceGetControllerPowerState", ["int", []]); $.IOBluetoothPreferenceGetControllerPowerState()'
sudo profiles list
launchctl print gui/$(id -u)/local.bluetooth-off | grep -E "state =|pid ="
sudo launchctl print system/local.p2p-off | grep -E "state =|last exit"
ifconfig awdl0 | grep flags; ifconfig llw0 | grep flags; ifconfig ap1 | grep flags
networksetup -listallnetworkservices
sudo launchctl print-disabled system | grep -iE "cups|netbios|smbd|screensharing|sshd|AEServer"
pgrep -l netbiosd
scutil --get ComputerName; scutil --get LocalHostName; scutil --get HostName
sudo lsof -nP -iTCP -sTCP:LISTEN
```

| 检查项 | 期望结果 | 2026-11-24 重启后 |
|---|---|---|
| pf 状态 | Status: Enabled | ✅ |
| 主规则集 | anchor "user.block" all | ✅ |
| anchor 规则 | 14 条 | ✅ |
| 应用防火墙 | 阻止所有传入连接，隐身模式开启 | ✅ |
| Wi-Fi 电源 | On | ✅ |
| 蓝牙控制器 | 0 | ✅ |
| 蓝牙描述文件 | 列出 local.disable-bluetooth | ⚠️ 待确认 |
| local.bluetooth-off | state = running | ✅ |
| local.p2p-off | last exit code = 0 | ✅ |
| awdl0 / llw0 / ap1 | flags 中没有 UP | ✅ |
| 网络服务 | 只有 Wi-Fi 没有 * | ✅ |
| 已禁用服务 | 6 项都是 disabled | ✅ |
| netbiosd | 无输出 | ✅ |
| 机器名 | pc / pc / pc | ✅ |
| TCP 监听端口 | 无 | ✅ |

---

## 五、恢复单项功能

蓝牙（先在 设备管理 中删除 Disable Bluetooth）：

```
launchctl bootout gui/$(id -u)/local.bluetooth-off
rm ~/Library/LaunchAgents/local.bluetooth-off.plist
osascript -l JavaScript -e 'ObjC.import("IOBluetooth"); ObjC.bindFunction("IOBluetoothPreferenceSetControllerPowerState", ["void", ["int"]]); $.IOBluetoothPreferenceSetControllerPowerState(1)'
```

AWDL / 隔空投送：

```
sudo launchctl bootout system/local.p2p-off
sudo rm /Library/LaunchDaemons/local.p2p-off.plist
```

打印：

```
sudo launchctl enable system/org.cups.cupsd
```

查找我的 Mac（睡眠时保持连接）：

```
sudo pmset -a tcpkeepalive 1
```

---

## 六、已知情况与记录

- `cupsctl: Unable to connect to server`：cupsd 没有运行时出现，无害；cupsd 已被禁用
- `com.apple.smb.server` 写入失败：受系统保护；smbd 和 netbiosd 都已禁用，不影响
- 远程管理 (ARD) 已关闭。如果当时在"终端想要管理这台电脑"的弹窗里点了允许，到 隐私与安全性 里撤销终端的这项权限
- Zen 连接 `lb-140-82-11x-26-iad.github.com:443`：这是 Zen 自己发起的出站连接，Zen 已验证为 Mauro Baladés 签名。`lb-…-iad.github.com` 是 GitHub 官方的反向解析名，不是攻击
- `Helper (Aperitif)`：Chromium 157 及以上版本的新网络辅助进程，属于正常现象
- Brave Browser 已于 2026-11-24 完全删除，包括钥匙串里的 Brave Safe Storage 和所有残留文件

## 七、局限

- mDNSResponder 受 SIP 保护，关不掉；它的广播已用 NoMulticastAdvertisements 关闭，出站 5353 也被 pf 封掉
- awdl0 由 local.p2p-off 每 10 秒关一次，中间最多可能有几秒处于开启状态
- 蓝牙循环用的是苹果私有 API，系统更新后可能失效，主要靠描述文件
- N1 芯片的 Thread 没有开关：不要在这台 Mac 上设置"家庭"App 和 Matter 配件
- 恢复模式 (recoveryOS) 下以上设置都不生效
- 只靠软件无法保证射频绝对静默

## 八、路由器待办（FRITZ!Box）

- [ ] 恢复出厂设置（不要用旧备份恢复）
- [ ] 升级到最新 FRITZ!OS
- [ ] 新管理员密码、新 Wi-Fi 密码，WPA2/WPA3
- [ ] 关闭 WPS
- [ ] 关闭端口转发、MyFRITZ!/远程访问，删除陌生设备和 VPN 用户
- [ ] 路由器修好之前，Mac 使用加密 DNS（DoH/DoT）
