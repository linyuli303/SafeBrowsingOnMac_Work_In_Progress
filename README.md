# Mac 装机后断网设置（虚拟机安装前）

> 整理日期：2026-11-22 ｜ 设备：MacBook Air（M5，Apple N1 无线芯片）｜ 系统：macOS 26

## 使用前须知

- 每次打开终端，先单独执行 `setopt interactivecomments`。macOS 默认的 zsh 不认交互输入里的 `#` 注释，不执行这一行的话，注释行会报错。
- 粘贴含 `sudo` 的整段命令之前，先单独执行一次 `sudo -v` 并输入密码。否则 sudo 在粘贴过程中弹出密码提示时，会把后面粘贴进来的命令当成密码吞掉。
- 只用代码块的复制按钮复制。如果要存成文本文件：在“文本编辑”里选 Format → Make Plain Text，并关闭 Edit → Substitutions → Smart Quotes。
- 生成 plist 或描述文件后，先用 `plutil -lint` 检查，结果为 `OK` 再加载或安装。
- 带 `sudo` 的命令在管理员账户里执行；蓝牙相关的 `osascript` 命令不要加 `sudo`。

## 0. 装机与激活

1. DFU 恢复（在 Apple Store 做，或者用另一台 Mac）。
2. 激活必须联网：用可信网络（Apple Store 的网络或 iPhone 个人热点），不要用家里的 Wi-Fi。
3. 设置助理：跳过 Apple ID；关闭定位服务、分析与改进；拒绝“用 iPhone 设置”和共享 Wi-Fi 密码。
4. 检查软件更新。
5. 开启锁定模式：System Settings → Privacy & Security → Lockdown Mode → Turn On，然后重启。

## 1. 断网前准备（仍在可信网络上）

如果不想在这台 Mac 上联网下载，就用另一台设备下载，再用 U 盘拷过来，校验步骤照样做。

需要准备：

- UTM.dmg（github.com/utmapp/UTM/releases）
- Ubuntu LTS ARM64 ISO，以及官方的 SHA256SUMS
- Firefox / Zen 安装包
- Little Snitch 安装包和规则备份（装好后直接导入）
- 本文档（纯文本）

### UTM 校验

```bash
shasum -a 256 ~/Downloads/UTM.dmg
hdiutil verify ~/Downloads/UTM.dmg
hdiutil attach ~/Downloads/UTM.dmg -readonly -nobrowse
codesign --verify --deep --strict --verbose=2 /Volumes/UTM/UTM.app
codesign -dv --verbose=4 /Volumes/UTM/UTM.app 2>&1 | grep -E "^Identifier|^Authority|^TeamIdentifier"
spctl -a -t exec -vv /Volumes/UTM/UTM.app
```

| 检查项 | 期望结果 |
|---|---|
| SHA256 | 与 GitHub 上该版本 UTM.dmg 旁边显示的 sha256 完全一致 |
| hdiutil verify | `VALID` |
| codesign --verify | `valid on disk`、`satisfies its Designated Requirement` |
| 签名者 | `Identifier=com.utmapp.UTM`；`Developer ID Application: Turing Software, LLC (WDNLXAD4W8)`；证书链到 `Apple Root CA` |
| spctl | `accepted`、`source=Notarized Developer ID` |

DMG 本身没有签名是正常的。任何一项对不上，就删除文件，不要打开。

全部通过后安装：

```bash
sudo ditto /Volumes/UTM/UTM.app /Applications/UTM.app
hdiutil detach /Volumes/UTM
codesign --verify --deep --strict --verbose=2 /Applications/UTM.app
```

### Ubuntu ISO 校验

```bash
shasum -a 256 ~/Downloads/ubuntu-*-arm64.iso
```

结果必须和 ubuntu.com 上同一版本的 SHA256SUMS 完全一致。

### 其他 App 校验

```bash
codesign --verify --deep --strict --verbose=2 "/Applications/App名称.app"
codesign -dv --verbose=4 "/Applications/App名称.app" 2>&1 | grep -E "^Authority|^TeamIdentifier"
spctl -a -t exec -vv "/Applications/App名称.app"
```

签名者应为对应的开发商（Firefox：Mozilla Corporation；Little Snitch：Objective Development Software GmbH；Zen：与官网说明一致），并显示 `Notarized Developer ID`。

## 2. 断网脚本（装机后第一步）

先单独执行 `setopt interactivecomments` 和 `sudo -v`，再粘贴下面整段。

```bash
# 1. 无线电断电：Wi-Fi、蓝牙（蓝牙这一行不要加 sudo）
networksetup -setairportpower en0 off
osascript -l JavaScript -e 'ObjC.import("IOBluetooth"); ObjC.bindFunction("IOBluetoothPreferenceSetControllerPowerState", ["void", ["int"]]); $.IOBluetoothPreferenceSetControllerPowerState(0)'

# 2. 禁用所有网络服务（重启后仍保持）
networksetup -listallnetworkservices | tail -n +2 | sed 's/^\*//' | while read -r s; do sudo networksetup -setnetworkserviceenabled "$s" off; done

# 3. 关闭除 lo0 外的所有网络接口（重启后失效：重新执行，或用第 4.1 节的开机任务）
for i in $(ifconfig -l); do [ "$i" != "lo0" ] && sudo ifconfig "$i" down; done

# 4. AirDrop
defaults write com.apple.sharingd DiscoverableMode -string "Off"
killall sharingd

# 5. Bonjour 广播
sudo defaults write /Library/Preferences/com.apple.mDNSResponder.plist NoMulticastAdvertisements -bool YES
sudo killall mDNSResponder

# 6. 接力 / AirPlay 接收器
defaults -currentHost write com.apple.coreservices.useractivityd ActivityAdvertisingAllowed -bool no
defaults -currentHost write com.apple.coreservices.useractivityd ActivityReceivingAllowed -bool no
defaults -currentHost write com.apple.controlcenter AirplayRecieverEnabled -bool false

# 7. 共享与远程服务
sudo launchctl disable system/com.openssh.sshd
sudo launchctl disable system/com.apple.AEServer
sudo launchctl disable system/com.apple.screensharing
sudo launchctl disable system/com.apple.smbd
sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -deactivate -stop

# 8. CUPS 打印服务
sudo cupsctl --no-share-printers --no-remote-any --no-remote-admin
sudo launchctl disable system/org.cups.cupsd

# 9. NetBIOS（只执行一次；unload 若提示 SIP 不允许，跳过即可）
echo "[default]" | sudo tee -a /etc/nsmb.conf
echo "protocol_vers_map=6" | sudo tee -a /etc/nsmb.conf
echo "port445=no_netbios" | sudo tee -a /etc/nsmb.conf
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.netbiosd.plist

# 10. 机器名去掉身份信息（名称可自定，不要包含真名和型号）
sudo scutil --set ComputerName "pc"
sudo scutil --set LocalHostName "pc"
sudo scutil --set HostName "pc"
sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.smb.server NetBIOSName -string "PC"

# 11. 关闭入网检测（captive.apple.com）
sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.captive.control Active -bool false

# 12. 睡眠时联网
sudo pmset -a womp 0
sudo pmset -a powernap 0
sudo pmset -a tcpkeepalive 0
```

## 3. 蓝牙全局关闭（系统级配置描述文件）

对所有账户生效，包括登录之前。

```bash
cat > ~/Desktop/disable-bluetooth.mobileconfig <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>PayloadContent</key>
  <array>
    <dict>
      <key>PayloadType</key>
      <string>com.apple.ManagedClient.preferences</string>
      <key>PayloadIdentifier</key>
      <string>local.disable-bluetooth.mcx</string>
      <key>PayloadUUID</key>
      <string>$(uuidgen)</string>
      <key>PayloadVersion</key>
      <integer>1</integer>
      <key>PayloadContent</key>
      <dict>
        <key>com.apple.MCXBluetooth</key>
        <dict>
          <key>Forced</key>
          <array>
            <dict>
              <key>mcx_preference_settings</key>
              <dict>
                <key>DisableBluetooth</key>
                <true/>
              </dict>
            </dict>
          </array>
        </dict>
      </dict>
    </dict>
  </array>
  <key>PayloadDisplayName</key>
  <string>Disable Bluetooth</string>
  <key>PayloadIdentifier</key>
  <string>local.disable-bluetooth</string>
  <key>PayloadScope</key>
  <string>System</string>
  <key>PayloadType</key>
  <string>Configuration</string>
  <key>PayloadUUID</key>
  <string>$(uuidgen)</string>
  <key>PayloadVersion</key>
  <integer>1</integer>
</dict>
</plist>
EOF
plutil -lint ~/Desktop/disable-bluetooth.mobileconfig
open ~/Desktop/disable-bluetooth.mobileconfig
```

安装：System Settings → General → Device Management → 双击 “Disable Bluetooth” → Install，输入管理员密码。

检查（应输出 `true`）：

```bash
osascript -l JavaScript -e '$.NSUserDefaults.alloc.initWithSuiteName("com.apple.MCXBluetooth").objectForKey("DisableBluetooth").js'
```

## 4. 强制保持（可选）

### 4.1 网络接口：开机自动关闭，之后每 60 秒重复一次

```bash
sudo tee /Library/LaunchDaemons/local.radios-off.plist > /dev/null <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>local.radios-off</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/sh</string>
    <string>-c</string>
    <string>/usr/sbin/networksetup -setairportpower en0 off; for i in $(/sbin/ifconfig -l); do [ "$i" != lo0 ] &amp;&amp; /sbin/ifconfig "$i" down; done</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>StartInterval</key>
  <integer>60</integer>
</dict>
</plist>
EOF
sudo chown root:wheel /Library/LaunchDaemons/local.radios-off.plist
sudo chmod 644 /Library/LaunchDaemons/local.radios-off.plist
plutil -lint /Library/LaunchDaemons/local.radios-off.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/local.radios-off.plist
```

其中的 `&amp;&amp;` 是 XML 的写法，是故意这样写的，不要改。

### 4.2 蓝牙：常驻循环，每 0.2 秒检查一次（不要加 sudo）

```bash
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
  if ($.IOBluetoothPreferenceGetControllerPowerState() !== 0) {
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
plutil -lint ~/Library/LaunchAgents/local.bluetooth-off.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.bluetooth-off.plist
```

如果弹出“已添加后台项目”，在 System Settings → General → Login Items 里保持允许。

测试：

```bash
launchctl print gui/$(id -u)/local.bluetooth-off | grep -E "state =|pid ="
kill $(pgrep -f IOBluetoothPreference); sleep 2; launchctl print gui/$(id -u)/local.bluetooth-off | grep -E "state =|pid ="
```

两次都应该是 `state = running`，第二次的 `pid` 变成新的数字。在控制中心里手动打开蓝牙，它应该立刻被关回去。重启后再查一次，确认它会自动启动。

## 5. 检查清单

```bash
networksetup -getairportpower en0
system_profiler SPBluetoothDataType SPAirPortDataType | grep -iE "state|status"
osascript -l JavaScript -e 'ObjC.import("IOBluetooth"); ObjC.bindFunction("IOBluetoothPreferenceGetControllerPowerState", ["int", []]); $.IOBluetoothPreferenceGetControllerPowerState()'
osascript -l JavaScript -e '$.NSUserDefaults.alloc.initWithSuiteName("com.apple.MCXBluetooth").objectForKey("DisableBluetooth").js'
ifconfig -lu
route -n get default
networksetup -listallnetworkservices
sudo launchctl print-disabled system | grep -iE "cups|netbios|smbd|screensharing|sshd|AEServer"
ps aux | grep -i netbiosd | grep -v grep
scutil --get ComputerName; scutil --get LocalHostName; scutil --get HostName
```

| 检查项 | 期望结果 |
|---|---|
| Wi-Fi 电源 | `Wi-Fi Power (en0): Off` |
| 蓝牙 / Wi-Fi 状态 | 蓝牙 `State: Off`、`Address: NULL`；Wi-Fi `Status: Off` |
| 蓝牙控制器 | `0` |
| 蓝牙全局策略 | `true` |
| 已启用的接口 | 只有 `lo0` |
| 默认路由 | `not in table` |
| 网络服务 | 每一项前面都有 `*` |
| 已禁用的服务 | 每一项都是 `disabled` |
| netbiosd | 无输出（如果 unload 被 SIP 拦截，这里会有输出） |
| 机器名 | 都是 `pc` |

每次 macOS 更新后，都把这一节重新检查一遍。

## 6. 装完虚拟机后恢复联网

虚拟机要联网之前，必须先移除 4.1 的开机任务，否则它会把虚拟机的网络接口也关掉。

先单独执行 `setopt interactivecomments` 和 `sudo -v`，再粘贴：

```bash
# 移除网络开机任务（如已安装）
sudo launchctl bootout system/local.radios-off
sudo rm /Library/LaunchDaemons/local.radios-off.plist

# 打开 Wi-Fi（服务名以 networksetup -listallnetworkservices 显示的为准，德语系统为 "WLAN"）
sudo networksetup -setnetworkserviceenabled "Wi-Fi" on
networksetup -setairportpower en0 on
sudo ifconfig en0 up
```

联网时在系统设置里：

- 加入网络后：Wi-Fi → 该网络的 Details → Private Wi-Fi Address → Rotating
- Wi-Fi → Ask to join networks → Off；Ask to join hotspots → Never
- 不要保存隐藏网络；删除用不到的已知网络
- Privacy & Security → Location Services → Off
- 家里的路由器恢复出厂、更新固件、换掉 Wi-Fi 密码之前，只用手机热点或其他可信网络

蓝牙仍保持关闭（第 3、4.2 节）。

## 7. 需要时恢复单项功能

蓝牙：

1. System Settings → General → Device Management → 选中 “Disable Bluetooth” → 删除
2. 执行：

```bash
launchctl bootout gui/$(id -u)/local.bluetooth-off
rm ~/Library/LaunchAgents/local.bluetooth-off.plist
osascript -l JavaScript -e 'ObjC.import("IOBluetooth"); ObjC.bindFunction("IOBluetoothPreferenceSetControllerPowerState", ["void", ["int"]]); $.IOBluetoothPreferenceSetControllerPowerState(1)'
```

打印：

```bash
sudo launchctl enable system/org.cups.cupsd
```

## 8. 局限

- mDNSResponder 关不掉：它受 SIP 保护，而且本身就是 macOS 的 DNS 解析器。离线时，无线电断电加上接口关闭，它无法发出任何信号；联网时，`NoMulticastAdvertisements` 让它不再广播这台 Mac。
- netbiosd 的 unload 在部分系统版本上会被 SIP 拦截；这种情况下靠第 10 步的改名，让它广播出去的不是你的信息。
- Thread（N1 芯片）没有单独的开关：不要在这台 Mac 上设置“家庭”App 和 Matter 配件。
- 4.2 的蓝牙循环用的是苹果私有 API，系统更新后可能失效，所以要重做第 5 节的检查。4.2 是用户级任务，这个账户下运行的任何程序都能停掉它；主力是第 3 节的全局描述文件。
- 恢复模式（recoveryOS）是独立的系统，上面所有设置在那里都不生效。
- 软件层面最多做到“系统让无线电断电”。要绝对保证不发任何信号，只能用无线模块可以物理拆除的硬件；要独立验证，只能靠外部射频检测（用另一台设备做 BLE 扫描、Wi-Fi 监听模式抓包）。
