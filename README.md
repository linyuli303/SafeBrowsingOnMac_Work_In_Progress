# Zen 浏览器重装与安全加固记录

日期：2026-11-24
设备：MacBook Air（M5），Little Snitch + pf 防火墙
背景：住处 Wi-Fi 疑似被攻破（路由器非本人管理）

---

## 1. 起因：Udemy 视频无法播放

- 视频分片能正常下载（200），但 DRM 授权请求被拒：
  - `POST www.udemy.com/media-license-server/validate-auth-token` → **403**
  - 返回内容：`Widevine license generation failed >> Verified media path has been tampered`
- 排除项：
  - Widevine 已安装：版本 4.10.3112.0，11 月 21 日首次安装 Zen 时就已下载
  - 放行 Sift 反欺诈服务（`cdn.sift.com`、`hexagon-analytics.com`）后仍然 403
  - 被拦的其他请求都是统计/监控（litix.io、datadog、sentry、googletagmanager），与播放无关
- 结论：**VMP（Verified Media Path）认证失败**，是浏览器层面的问题，与防火墙规则无关

---

## 2. 签名排查（codesign）

### 2.1 严格模式检查失败

```
codesign --verify --deep --strict --verbose=2 /Applications/Zen.app
```

- 报错：`resource fork, Finder information, or similar detritus not allowed`
- 原因：应用包里的文件带有 `com.apple.FinderInfo` 扩展属性（xattr）
- 每次报错的文件不同（libmozavutil、libgkcodecs、gpu-helper、updater 等），因为 codesign 遇到第一个就停止

### 2.2 逐个文件检查开发者签名

- **所有文件** `TeamIdentifier=9V5K9TP787`（Zen 官方开发者 Mauro Baladés）
- **所有文件**都带 FinderInfo

### 2.3 尝试清除 xattr

```
sudo xattr -cr /Applications/Zen.app
```

- 全部 `Operation not permitted`：macOS「应用管理（App Management）」保护阻止修改应用包，连 sudo 也不行
- 这说明其他程序同样无法篡改 Zen 的应用包

### 2.4 VMP 签名文件

- 完整文件列表中**没有任何 `.sig` 文件**（Firefox 官方版的 VMP 签名以 `.sig` 文件形式存放）
- 推断：Zen 这个版本没有 Widevine VMP 签名 → Udemy 拒绝授权

### 2.5 手动删除库文件（误操作）

- 手动删除应用包内的库文件后，签名验证结果变为：`code object is not signed at all`
- 应用包已损坏，不能再运行

---

## 3. 下载官方安装包并验证

### 3.1 用 Brave 下载（Little Snitch 临时规则）

- Brave：Allow / Outgoing / TCP 443
  - `zen-browser.app`
  - `github.com, githubassets.com, githubusercontent.com`
- 下载完成后**单独取消勾选这两条规则**（不要停用整个 Zen 组，组里有 Zen 的 Deny Any 兜底规则）

### 3.2 验证 dmg 文件

```
codesign --verify --deep --strict --verbose=2 ~/Downloads/zen.macos-universal.dmg
codesign -dv ~/Downloads/zen.macos-universal.dmg
spctl -a -t open --context context:primary-signature -v ~/Downloads/zen.macos-universal.dmg
find ~/Downloads/zen.macos-universal.dmg -xattrname com.apple.FinderInfo
```

| 检查项 | 结果 |
|---|---|
| 签名 | `valid on disk`，`satisfies its Designated Requirement` |
| 开发者 | `TeamIdentifier=9V5K9TP787` |
| 签名时间 | 2026-11-18 06:12:04 |
| 苹果公证 | `accepted`，`source=Notarized Developer ID` |
| dmg 上的 FinderInfo | 无 |

注：`spctl --type execute` 对 dmg 显示 `rejected (does not seem to be an app)` 是正常的，dmg 要用 `-t open` 检查。

### 3.3 关键发现：FinderInfo 来自官方打包

- 挂载后的 dmg 是**只读**的，但里面的 `Zen.app/Contents/MacOS/libmozavcodec.dylib` **已经带有 FinderInfo**
- **结论**：FinderInfo 是 Zen 官方打包时留下的，**不是恶意注入**，也不是安装过程造成的
- FinderInfo 是固定 32 字节的 Finder 元数据（标签、隐藏标记等），不属于文件内容，也装不下有意义的代码
- 这个版本的 Zen 永远过不了 `--strict` 检查。今后验证 Zen 用：

```
codesign --verify --deep --verbose=2 /Applications/Zen.app
spctl --assess --type execute --verbose /Applications/Zen.app
```

  - 不带 `--strict` 时仍会逐个核对每个文件内容的哈希值
  - 预期：`valid on disk` + `accepted source=Notarized Developer ID`

---

## 4. 彻底删除旧 Zen

### 4.1 找出所有相关文件

```
find ~/Library /Library -maxdepth 3 \( -iname "zen" -o -iname "*zen-browser*" -o -iname "Zen.app" \) 2>/dev/null
ls ~/Library/Application\ Support/Mozilla ~/Library/Caches/Mozilla 2>/dev/null
```

找到的：
- `~/Library/Application Support/zen`（配置文件夹：扩展、设置、Cookie、Widevine 等）
- `~/Library/Preferences/app.zen-browser.zen.plist`
- `~/Library/Caches/zen`
- `~/Library/Caches/Mozilla/updates`

注：运行时弹出「Terminal would like to access data from other apps」→ 选 **Don't Allow**（Zen 不是沙盒应用，不需要这个权限）

### 4.2 删除

- `/Applications/Zen.app`：在 **Finder** 里拖进废纸篓并清空（终端会被应用管理保护挡住）

```
defaults delete app.zen-browser.zen 2>/dev/null
rm -rf "$HOME/Library/Application Support/zen"
rm -rf ~/Library/Caches/zen
rm -rf ~/Library/Caches/Mozilla
rm -f ~/Library/Preferences/app.zen-browser.zen.plist
```

⚠️ 教训：路径中有空格时必须加引号或在空格前加 `\`。
`rm -rf /Users/.../Application Support/zen` 不加引号时，会被当成两个路径（`…/Application` 和 `Support/zen`）。这次两个路径都不存在，所以没有误删，但这样写非常危险。

### 4.3 确认删干净

```
find ~/Library /Library -maxdepth 3 \( -iname "zen" -o -iname "*zen-browser*" -o -iname "Zen.app" \) 2>/dev/null
ls /Applications | grep -i zen
```

两条都没有输出 = 删干净。

---

## 5. 干净安装

1. 从挂载的 dmg 把 Zen 拖进「应用程序」
2. 用第 3.3 节的两条命令验证
3. 推出 dmg 后打开 Zen

新配置文件夹：
- `~/Library/Application Support/zen/Profiles/95xd2il2.Default (release)`
- 创建时间：2026-11-24 16:50:29
- `chrome/` 文件夹：同一时间创建，是 Zen 主题系统自动生成的（**待确认内容**：里面应只有 `.css` 文件，不应有 `.js`）
- `extensions/`：只有 `uBlock0@raymondhill.net.xpi`（17:31 安装）

---

## 6. uBlock Origin：从终端下载并核对哈希

Little Snitch 临时规则：Terminal（via curl）→ `addons.mozilla.org, addons.cdn.mozilla.net`（Hosts，TCP 443）

```
cd ~/Downloads
curl -sSL -o ublock.json "https://addons.mozilla.org/api/v5/addons/addon/ublock-origin/"
plutil -extract current_version.file.url raw -o - ublock.json
plutil -extract current_version.file.hash raw -o - ublock.json
curl -L -o ublock_origin.xpi "（上一步得到的下载地址）"
shasum -a 256 ublock_origin.xpi
```

- 算出的哈希与 API 返回的 `sha256:` 值一致后再安装
- 安装：`about:addons` → 齿轮 → Install Add-on From File
- 确认开发者为 Raymond Hill；Zen 会自动验证 Mozilla 签名
- 装好后删除 `ublock.json`、`ublock_origin.xpi`，停用终端的临时规则

### My filters 恢复

- 备份文件：`~/Downloads/ZEN/my-ublock-static-filters_2026-11-22_21.06.53.txt`
- 内容：YouTube 界面规则（隐藏 Shorts、首页推荐、右侧推荐、片尾推荐墙、暂停推荐、全屏推荐、搜索结果里的推荐、片尾卡片，屏蔽字幕下载）
- 已审查：全是自己写的元素隐藏规则 + 1 条字幕网络规则，无脚本注入规则，无外部域名
- 导入：Dashboard → My filters → Import and append → Apply changes

### 过滤列表更新

- 因为全局封锁了 `github.io`、`githubusercontent.com`，需要给 Zen 放行备用地址：`ublockorigin.pages.dev`、`cdn.jsdelivr.net`

---

## 7. Zen 的 about:config 设置

| 设置项 | 值 | 原因 |
|---|---|---|
| `network.http.http3.enable` | `true` | 关掉后视频出错，保持开启；Little Snitch 需要保留 UDP/QUIC 规则 |
| `network.http.http2.coalesce-hostnames` | `false` | 防止请求借用其他域名的连接，绕过 Little Snitch 的按域名规则 |
| `browser.safebrowsing.downloads.remote.enabled` | `true` | 安全优先：下载文件时做云端恶意软件检查 |
| `privacy.clearOnShutdown_v2.cookiesAndStorage` | `true`（建议） | 退出时清除 Cookie，降低登录 Cookie 被盗的风险；需要保持登录的网站加入例外 |
| `privacy.sanitize.sanitizeOnShutdown` | `true` | 上一项生效的前提 |
| `browser.contentblocking.category` | `strict` | 跟踪保护严格模式（已确认） |
| `dom.security.https_only_mode` | `true` | 只走 HTTPS，防止被攻破的网络降级/篡改 HTTP |
| `media.peerconnection.enabled` | `false` | 关闭 WebRTC，避免泄露本机 IP |
| `network.captive-portal-service.enabled` | `false` | 不发送不加密的 80 端口检测请求 |
| `app.normandy.enabled` | `false` | Zen 默认已关闭（远程实验系统） |
| `app.shield.optoutstudies.enabled` | `false` 🔒 | Zen 默认已锁定 |
| `app.normandy.api_url` | 空 🔒 | Zen 默认已锁定 |

- Zen 自带的 DNS over HTTPS：**关闭**。开启后网页打不开（DoH 服务器不在白名单里），并且会让 Little Snitch 看不到域名
- `user.js` 位置：配置文件夹的最外层（和 `prefs.js` 同级），不要放进 `chrome/`

---

## 8. Little Snitch 规则调整

### Zen

- **YouTube 组**
  - TCP 443（Domains）：`youtube.com, googlevideo.com, ytimg.com, ggpht.com, youtube-nocookie.com, lh3.googleusercontent.com, gstatic.com`
  - UDP 443 / QUIC：保留（HTTP/3 开着），建议与 TCP 规则的域名保持一致
  - `accounts.google.com`：**Ask**（只在需要登录时放行）
- **Safe Browsing 组**（Hosts，TCP 443）
  - `firefox.settings.services.mozilla.com`
  - `firefox-settings-attachments.cdn.mozilla.net`
  - `shavar.services.mozilla.com`
  - `safebrowsing.googleapis.com`
  - `sb-ssl.google.com`
- **Udemy**：Zen 无法通过 VMP 认证，Udemy 相关规则和临时规则可删除；Udemy 改用 Brave 或官方 Firefox
- **Widevine 更新**：`gvt1.com`（Domains）、`aus5.mozilla.org`（Hosts）

### 全局封锁（计划）

- Any Process / Owner: System / Deny / Outgoing
- Domains：`cloudfront.net, githubusercontent.com, github.io`
- Priority: **High** 才能压过各应用的放行规则

### DNS 加密

- 开启 Little Snitch 自带的 **DNS 加密**（整台 Mac 生效）
- 开启后，之前出现的 `*.ip-addr.inexio.net`（运营商机房里的 Google 视频缓存节点）等连接消失

### 系统安全组件（建议放行）

- `syspolicyd`（Gatekeeper）→ `api.apple-cloudkit.com`
- Little Snitch Software Update → `obdev.at`
- 检查 XProtect 是否在更新：

```
system_profiler SPInstallHistoryDataType | grep -A4 XProtect
```

---

## 9. 备份

```
P="$HOME/Library/Application Support/zen/Profiles/95xd2il2.Default (release)"
B="$HOME/Downloads/ZEN/zen-backup-$(date +%Y-%m-%d_%H%M)"
mkdir -p "$B"
cp -p "$P/prefs.js" "$P/user.js" "$B/"
ls -la "$B"
```

位置：`~/Downloads/ZEN/zen-backup-日期_时间/`

---

## 10. 待办

- [ ] 检查 `chrome/` 文件夹内容：`ls -laR "$P/chrome"`、`find "$P/chrome" -type f -name "*.js"`
- [ ] 观察 `mDNSResponder → 192.168.178.1:53` 规则的使用次数，不再增长后停用
- [ ] 停用 Claude 桌面应用直连 `192.168.178.1:53` 的规则
- [ ] 给 Zen 放行 uBlock 过滤列表的备用更新地址
- [ ] 确认 Brave 下载 Zen 用的临时规则已取消勾选
- [ ] 在 Udemy 退出登录再重新登录（排查时会话 Cookie 被贴出过）
- [ ] 用 Brave 或官方 Firefox 看 Udemy
- [ ] 运行防火墙检查命令（应用防火墙、pf、监听端口、共享服务、IPv6），并导出最新的 Little Snitch 备份进行审查
- [ ] 长期：Mac 只连 Mango，Mango 上关 IPv6 + VPN
- [ ] 可选：向 Zen 的 GitHub 报告官方 dmg 里的文件带 `com.apple.FinderInfo`，导致 `codesign --strict` 失败
