# ACL4SSR（私有分支）

基于 [`cmliu/ACL4SSR`](https://github.com/cmliu/ACL4SSR) 的私有规则库，在其模板结构上扩展了 AI / TikTok / X / Muse / Gemini / Google Play / FCM 七类规则。

- 上游 commit：`f111a3169146`（2026-11-03）
- 上游历史为每周自动提交的二进制（GeoLite2、adobe.list），已压缩为单次 import 提交，不保留
- 上游原有文件全部保留未改动，新规则以独立文件追加，互不影响

---

## 一、新增了什么

### `Clash/*.list` — 经典格式（`behavior: classical`）

每行是一条完整规则，如 `DOMAIN-SUFFIX,tiktok.com`。

| 文件 | 条数 | 覆盖 |
|---|---:|---|
| `TikTok.list` | 42 | 主站 + App API + CDN + 直播 + Shop + 美/欧区 |
| `X.list` | 28 | x.com + twitter.com + twimg 图片/视频 CDN + t.co |
| `Muse.list` | 4 | `*.muse.ai` 全子域 + `meta.ai` |
| `Gemini-Full.list` | 49 | Gemini 网页 + App 私有端点 + NotebookLM + AI Studio |
| `GooglePlay-Full.list` | 22 | Play API + 图片 + **APK 下载 CDN gvt1/2/3** + 设备注册 |
| `GoogleFCM.list` | 17 | FCM 推送全部 mtalk 节点 |
| `AI-All.list` | 213 | ChatGPT / Claude / Grok / Perplexity / Copilot 等全量 |
| `Meta.list` | 24 | Facebook / Instagram / Messenger / Threads / WhatsApp / Meta AI / Muse AI |
| `TaiwanMedia.list` | 15 | 台湾流媒体全集：LiTV / 4GTV / OFiii / 巴哈姆特 / Hami Video / KKTV / friDay / LINE TV / myVideo / CatchPlay |

> `Gemini-Full.list` 取代上游 `Gemini.list`（13 条，2025-06 停更）。
> `Google.list` 合并取代上游 `PlayStore.list`（6 条）+ 原 `X.list` + 原 `GooglePlay-Full.list`。
> 旧文件保留，不删，方便回退。

### `Clash/domain/*.yaml` — domain 格式（`behavior: domain`）

同样七份数据，换成 `payload:` 列表。**domain 行为比 classical 匹配更快，推荐用这个。**

### `yaml/clash-full.yaml` — 开箱即用配置

**在你现有 misub 配置上做加法，不是另起一套。** 分组、规则顺序、DNS 全部沿用你原来那份：

| 项目 | 原有 | 变动 | 现在 |
|---|---:|---|---:|
| 策略组 | 27 | +4（🎵 TikTok / Google / 📱 Meta / 📺 台湾媒体）<br>-8（奈飞 / 哔哩哔哩 / 微软×3 / 游戏平台 / 网易音乐 / 巴哈姆特） | **23** |
| rule-provider | 41 | +8 / -13 | **36** |
| 规则 | 43 | +8 / -13 | **38** |

- `🤖 OpenAi` 组复用，Muse / Gemini / AI-All 三条规则都指向它
- `📢 谷歌FCM` 组复用，只换用 12 条完整版 provider（原 `googlefcm_6` 只有部分节点）
- 新增的 8 条规则**放在 rules 最前面**，先于 `playstore_0`，否则 Play 会被老规则抢走
- 已删除的 8 个组连同它们的 13 个 provider 和 13 条规则一起移除，无悬空引用
- `proxies:` 是 9 条占位节点，**name 与你原配置完全一致**（`🇭🇰 HK- DMIT` / `🇯🇵 JP-Vultr` 等），
  只改 `server` / `port` / 认证字段即可，别改 name——6 个地区 `load-balance` 组靠它解析

---

## 二、怎么用

### 方式 A：直接用 `yaml/clash-full.yaml`

把你自己的 9 条节点填进 `proxies:`（name 保持不动），导入 OpenClash / Clash Verge。
48 个 rule-provider 全部指向公网直链，无需额外配置。

### 方式 B：只摘规则片段

把 `rule-providers` 和 `rules` 两段粘进你自己的 yaml。**顺序必须这样，放 `rules:` 最前面：**

```yaml
rules:
  - RULE-SET,google_fcm,📢 谷歌FCM
  - RULE-SET,google_play,▶️ 谷歌商店
  - RULE-SET,tiktok,🎵 TikTok
  - RULE-SET,x_twitter,🐦 推特X
  - RULE-SET,muse,🤖 OpenAi
  - RULE-SET,gemini,🤖 OpenAi
  - RULE-SET,ai_all,🤖 OpenAi
  # ↓ 你原有的规则不动
```

FCM 必须排在 Play 前面，否则推送流量会被商店规则抢走。

### 方式 C：用本仓库自己的文件（私有仓库需 token）

```yaml
rule-providers:
  tiktok:
    type: http
    behavior: domain          # 对应 Clash/domain/TikTok.yaml
    url: "https://<TOKEN>@raw.githubusercontent.com/liuhui19994/ACL4SSR/main/Clash/domain/TikTok.yaml"
    path: ./ruleset/tiktok.yaml
    interval: 86400
    proxy: DIRECT
```

---

## 三、踩过的坑（改配置前先看这里）

**1. `behavior` 和文件格式必须配对，否则静默 0 条规则。**

| 文件内容 | behavior | 额外字段 |
|---|---|---|
| `DOMAIN-SUFFIX,tiktok.com` 纯文本 | `classical` | **必须写 `format: text`** |
| `payload: ['+.tiktok.com']` | `domain` | 不写（默认 yaml） |

`behavior: classical` 配 `.list` 纯文本时如果不写 `format: text`，mihomo 会报 `file must have a 'payload' field`；更糟的情况是不报错但 `ruleCount` 为 0，规则完全失效。

**2. rule-provider 下载会被代理规则劫持。**

provider 拉取走的是 mihomo 自己的规则匹配。如果代理组当时不可用，provider 静默失败，规则失效但没有任何提示。**必须加 `proxy: DIRECT`。**

**3. `respect-rules: true` 里也含 `rules:` 字样。** 用脚本改 yaml 时别拿 `"rules:"` 当锚点，会匹配到 `respect-rules:`。

**4. TikTok 光有域名规则没用。** 内容按 IP 归属地锁区，规则只决定走不走代理。要美区内容，落地必须是美国 IP。

**5. Muse 仅美/加开放，且解析到 Meta 自己的边缘 IP（`57.144.x`）。** 域名规则优先级高于 IP 规则，不会被 Facebook 的 GEOIP 规则误抓，但落地必须是美/加。

**6. Play 和 FCM 必须分开。** FCM 是常驻长连接，Play 是突发大流量，混一组会让推送被下载挤掉线。

**7. 台湾流媒体只能走台湾落地。** LiTV / 4GTV / OFiii 都按 IP 锁区，走非台湾节点会直接拒绝播放。`📺 台湾媒体` 组的默认第一项是 `🚀 节点选择`，用之前记得切到 `🇹🇼 台湾节点`。

**8. OFiii 后端就是 LiTV。** `ofiii.com` 页面里 `p-cdnstatic.svc.litv.tv` 出现 75 次，API 在 `fino.svc.litv.tv`。所以 `+.litv.tv` 一条通配同时覆盖了 OFiii 的 CDN 和 API，不必单独列。

**9. Play 更新吃流量。** `gvt1/2/3` 全走代理的话，小流量套餐会被应用更新吃掉大半。

---

## 四、目录

```
Clash/
  *.list              经典格式规则（上游原有 + 本分支新增 7 个）
  domain/*.yaml       domain 格式规则（本分支新增 7 个）
  config/*.ini        subconverter 模板（上游原有）
yaml/
  bdg.yaml            上游原有
  clash-full.yaml     本分支新增，开箱即用完整配置
CloudflareCIDR-main.py 上游原有
CloudflareCIDR.txt     上游原有
GeoLite2-Country.mmdb  上游原有
baipiao.txt            上游原有
```

`.github/workflows/` 下 4 个自动更新任务保留原样（adobe / baipiao / CloudflareCIDR / GeoLite2）。它们只改自己那几个文件，不碰新增规则。

---

## 五、数据来源

所有域名取自 [`MetaCubeX/meta-rules-dat`](https://github.com/MetaCubeX/meta-rules-dat) 的 geosite 分类，2026-11-04 版本：

```
category-ai-!cn   → AI-All
google-gemini     → Gemini-Full
tiktok            → TikTok
twitter           → X
google            → Google（Play + X + 账号 + Gmail）
googlefcm         → GoogleFCM
meta (muse 部分)  → Muse
```

`muse.ai` 的子域清单额外用 crt.sh 证书透明度日志核对过，共 29 个（`api` / `api1` / `api2` / `auth` / `files` / `live` / `vv` / `vv1-0-0` / `vv1-0-1` / `vv2-0-0` / `vv2-0-1` / `m00` / `m01` / `z` 等），`*.muse.ai` 一条通配全部覆盖。

---

## 两份配置怎么选

### `yaml/clash-full.yaml` —— 自建节点（回源 / CF SaaS）

`proxies:` 里是 **9 条占位节点**（`192.0.2.1–9`，TEST-NET-1 保留段，永远连不上）。
必须逐条把 `server` / `port` / `password` / `uuid` 换成你自己的节点，**`name` 一个字都不能改** ——
5 个地区 `load-balance` 组是靠名字精确匹配的，改了名字整个配置起不来。

### `yaml/clash-airport.yaml` —— 机场订阅（推荐现在用这个）

**没有 `proxies:` 段，一条占位节点都没有。** 节点由 `proxy-providers.airport` 从你的机场订阅实时拉取：

```yaml
proxy-providers:
  airport:
    type: http
    url: "https://PASTE-YOUR-AIRPORT-SUBSCRIPTION-URL-HERE"   # ← 只改这一行
    path: ./providers/airport.yaml
    interval: 86400
    health-check:
      enable: true
      url: http://www.gstatic.com/generate_204
      interval: 300
```

5 个地区组和 `☑️ 手动切换` 都改成 `use: [airport]`，靠 `filter:` 从机场节点里挑：

| 组 | filter |
|---|---|
| 🇭🇰 香港节点 | `港\|HK\|hk\|Hong Kong\|HongKong\|hongkong\|HKG` |
| 🇹🇼 台湾节点 | `台\|新北\|彰化\|TW\|Taiwan\|TPE\|KHH` |
| 🇯🇵 日本节点 | `日本\|川日\|东京\|大阪\|泉日\|埼玉\|沪日\|深日\|[^-]日\|JP\|Japan\|Tokyo\|NRT\|KIX` |
| 🇸🇬 狮城节点 | `新加坡\|坡\|狮城\|SG\|Singapore\|SIN` |
| 🇺🇲 美国节点 | `美\|波特兰\|达拉斯\|俄勒冈\|…\|US\|United States\|ATL\|…\|SJC` |

★ 如果你的机场节点命名跟上面这些关键词对不上（比如叫 `HK-01`、`HongKong Premium`），
对应地区组会是空的。改 `filter:` 的正则就行，不用动别的。

★ 机场节点名里带倍率后缀（`| 0.5x`）不影响，正则只做包含匹配。

---

## 📱 Meta 组为什么单独建，不并进 AI 组

### 事实：Muse AI 就是 Meta AI

```
muse.ai 证书    subject = O="Meta Platforms, Inc.", L=Menlo Park, CN=*.muse.ai
                SAN     = *.muse.ai, muse.ai
首页引用        www.facebook.com / www.meta.com
子域            api.muse.ai / auth.muse.ai / security.muse.ai
                → 全部 CNAME 到 star.c10r.facebook.com
A 记录          57.144.152.141  =  AS32934 Facebook, Inc.（物理新加坡）
```

所以 Muse AI app 登录、鉴权、拉 CDN 全走 Meta 自家边缘。**不给 Meta 建规则，Muse AI 登录必然出问题** —— OAuth 跳到 `facebook.com` 时会落到 `🐟 漏网之鱼`，出口节点跟 AI 组不一致，轻则提示异地登录，重则登录循环。

### 为什么不像你说的那样全部并进 AI 组

| 服务 | 并进 AI 组的问题 |
|---|---|
| **Facebook / Instagram** | 日常刷 IG 的流量会全走 AI 节点。机场的 AI 节点通常是**独享 / 高倍率**（3x–10x），刷半小时 IG 能烧掉几天的量。 |
| **推特 X** | X 对机房 IP 有强风控，登录会要求手机验证。AI 节点恰好是「被大量人用来跑 ChatGPT」的机房 IP，命中风控概率最高。**X 需要单独选一个家宽或原生 IP 节点**，跟 AI 需求相反。 |
| **谷歌商店** | Play 的 CDN 是 `gvt1.com` / `ggpht.com`，全球都有节点。并进 AI 组意味着**所有 App 下载都走 AI 节点**，一个游戏 2 GB 就是几倍流量。这是纯浪费。 |

### 为什么也不叫「美国媒体」

Meta 的边缘 IP `57.144.152.141` 实测物理位置在**新加坡**，是 AS32934 的 anycast。Facebook / Instagram 跟「美国」没有任何绑定关系。叫美国媒体会误导你以后调节点。

### 实际做法

新建 `📱 Meta`，**默认第一项是 `🤖 OpenAi`**：

```yaml
- name: 📱 Meta
  type: select
  proxies:
    - 🤖 OpenAi        # ← 默认，跟随 AI 组的出口，保证 OAuth 同地区
    - 🚀 节点选择
    - ♻️ 自动选择
    - 🇺🇲 美国节点
    - 🇸🇬 狮城节点
    - 🇭🇰 香港节点
    - 🇹🇼 台湾节点
    - 🇯🇵 日本节点
    - ☑️ 手动切换
    - DIRECT
```

这样两头都满足：**Muse app 登录不失败**（跟 AI 同出口），**平时想单独给 IG 换落地也能换**（不用连累 AI）。

### Meta 组覆盖的 24 个域名

```
facebook.com      facebook.net      fb.com         fb.me
fbcdn.net         fbsbx.com         accountkit.com
instagram.com     cdninstagram.com  instagr.am     ig.me
messenger.com     m.me
meta.com          meta.ai           ai.meta.com    muse.ai
threads.net       threads.com
whatsapp.com      whatsapp.net      wa.me
workplace.com     oculus.com
```

**没有抄 blackmatrix7 的 Facebook 列表（570 条）。** 那里面约 500 条是抢注的拼写错误域名——`faacebok.com`、`facebooik.com`、`facebopk.com`、`fcaebook.com` 这类，属于钓鱼和广告站，不是 Meta 的服务。收进来只会拖慢匹配、制造误伤。这里只保留真实在用的服务域名。

同理没收 `groups.com` / `i.org` / `klik.me` / `nextstop.com` / `parse.com` / `reactjs.org` 这些泛域名。

### 验证

`mihomo v1.19.14` 实测，把 `RULE-SET,meta` 指向 `REJECT`：

```
命中 22/22：
  www.facebook.com  m.facebook.com  graph.facebook.com  static.xx.fbcdn.net
  www.instagram.com  i.instagram.com  scontent.cdninstagram.com
  www.messenger.com  www.meta.com  meta.ai  ai.meta.com
  muse.ai  api.muse.ai  security.muse.ai
  www.threads.com  www.whatsapp.com  wa.me  www.workplace.com
  www.oculus.com  fb.com  ig.me  m.me

未误伤 6/6：
  www.google.com 200 / www.netflix.com 302 / twitter.com 301
  x.com 200 / play.google.com 302 / www.baidu.com 200
```

### 已并入 `Google` 组

`X.list`（28 条）和 `GooglePlay-Full.list`（22 条）已删除，内容合进 `Google.list`。详见下面「Google 组」一节。

---

## 🔍 Google 组：Play + X + 账号 + Gmail 合并

`▶️ 谷歌商店` 和 `🐦 推特X` 两个组合并成一个 `Google`，另补 Google 账号、登录、Gmail 邮箱。

### 为什么 X 能并进来

X 支持 Google 账号登录。OAuth 流程是：

```
X app  → accounts.google.com（选账号、输密码）
       → 回跳 x.com（拿 code 换 token）
```

如果两组分开，`accounts.google.com` 走 A 节点、`x.com` 走 B 节点，X 会判定「同一会话换了个国家」，轻则要手机验证，重则直接锁号。合并成一组，**出口天然一致**，这个坑就没了。

### 99 条构成

| 分节 | 条数 | 内容 |
|---|---|---|
| Google 账号 / 登录 / 身份 | 26 | `accounts.google.com` + 12 个国家版、`oauth2`、`securetoken`、`identitytoolkit`、`myaccount`、`people`、`passwords`、`pay` |
| Gmail / 邮箱 | 6 | `gmail.com`、`mail.google.com`、`inbox`、`googlemail.com`、`m.gmail.com`、`mail.googleusercontent.com` |
| Google 云服务 | 11 | `calendar` / `drive` / `photos` / `keep` / `meet` / `chat` / `maps` / `translate` / `store` / `voice` / `notification` |
| Google Play 商店 | 22 | Play API + 图片 + APK 下载 CDN `gvt0-9` + 设备注册 |
| Google 静态 / CDN / 短链 | 6 | `gstatic.com`、`g.co`、`goo.gl`、`withgoogle.com` |
| X / Twitter | 28 | `x.com` + `twitter.com` + `twimg` 图片/视频 CDN + `t.co` + `grok.com` |

### 故意不收的（会被前面的规则抢走，收了是浪费）

| 域名 | 归谁 |
|---|---|
| `fcm.googleapis.com` / `firebaseinstallations` / `mtalk.google.com` | 📢 谷歌FCM |
| `gemini.google.com` / `bard.google.com` / `aistudio.google.com` | 🤖 OpenAi |
| `youtube.com` / `ytimg.com` / `googlevideo.com` | 📹 油管视频 |
| `doubleclick` / `googleadservices` / `google-analytics` | 🛑 广告拦截 |

**特别注意 `accounts.youtube.com` 收进来了** —— 它是 YouTube 的**登录**入口，不是看视频。看视频走 `youtube.com`，仍然归 📹 油管视频。这样登录页和视频页出口一致，Google 不会判定异地。

### 也不收 `googleusercontent.com` 整域

这个域太宽：`blogger.googleusercontent.com`、`lh3.googleusercontent.com`（图片）、`mail.googleusercontent.com`、`play-lh.googleusercontent.com`……全收会把大量非 Google 服务的静态资源也拖进来。所以只按用途收 `mail.` / `play-lh.` / `play-games.` 三个子域。

### 组内顺序

```yaml
- name: Google
  type: select
  proxies:
    - 🚀 节点选择      # ← 默认，跟主出口一致
    - ♻️ 自动选择
    - 🇺🇲 美国节点
    - 🇸🇬 狮城节点
    - 🇭🇰 香港节点
    - 🇯🇵 日本节点
    - 🇹🇼 台湾节点
    - ☑️ 手动切换
    - DIRECT
```

默认 `🚀 节点选择` 而不是固定某地区——Google 账号对 IP 跳变极敏感，跟主出口保持一致最稳。

### 验证

`mihomo v1.19.14`，`RULE-SET,google` 指向 `REJECT`：

```
命中 32/32：
  账号  accounts / accounts.youtube / myaccount / myactivity / oauth2
        securetoken / identitytoolkit / www.googleapis / people
        passwords / pay
  邮箱  gmail.com / mail.google.com / inbox.google.com / googlemail.com
  云    calendar / drive / photos / meet
  Play  play.google.com / play.googleapis.com / play-lh.googleusercontent.com
        dl.google.com / redirector.gvt1.com / android.clients.google.com
  X     x.com / twitter.com / api.x.com / t.co / pbs.twimg.com
        video.twimg.com / grok.com

未误伤 10/10：
  www.youtube.com 200 / youtube.com 301 / i.ytimg.com 404
  fcm.googleapis.com 404 / gemini.google.com 200
  www.google.com 200 / www.netflix.com 302 / www.baidu.com 200
  www.facebook.com 200 / mtalk.google.com 000（纯 DIRECT 对照也是 000，域名本身不通）
```

### ⚠️ 合并后的一个代价

X 对机房 IP 的风控比 Google 更狠。合并后 X 和 Google 账号共用一个出口：

- 出口被 Google 标记 → X 也受影响
- 出口被 X 标记 → Google 登录也受影响

如果以后发现 X 频繁要手机验证，把 `Google` 组切到 `☑️ 手动切换` 里一个家宽/原生 IP 节点，别用机房 IP。

---

## 🤖 Gemini app 只命中一条 —— 根因与修法

### 现象

OpenClash 连接列表里，Gemini app 的请求：

```
growth-pa.googleapis.com:443           → proxygfwlist_33   ❌
notifications-pa.googleapis.com:443    → proxygfwlist_33   ❌
signaler-pa.googleapis.com:443         → proxygfwlist_33   ❌
robinfrontend-pa.googleapis.com:443    → gemini            ✅ 只有这一条
```

### 根因

`pa.googleapis.com` 是 Google 的 **Private Access** 域名族，几乎所有 Google 服务都在这个后缀下：

```
accounts-pa    oauth2-pa      mail-pa        gmail-pa
drive-pa       photos-pa      calendar-pa    people-pa
playstore-pa   cloudmessaging-pa   firebaseinstallations-pa
growth-pa      notifications-pa    signaler-pa
feedback-pa    semanticlocation-pa
alkalicore-pa  alkalimakersuite-pa  aisandbox-pa  geller-pa
robinfrontend-pa  proactivebackend-pa  notebooklm-pa  antigravity-pa
```

原来的 `Gemini-Full.list` 是**逐个具名**列的（49 条），只收了 AI 相关的十几个，`growth-pa` / `notifications-pa` / `signaler-pa` 三个漏了，于是落到第 29 位的 `proxygfwlist_33`。

### 修法：三层归属 + 调整规则顺序

**`pa.googleapis.com` 本身是空 zone**（没有 A 记录），但所有 `*.pa.googleapis.com` 都有。所以用 `DOMAIN-SUFFIX,pa.googleapis.com` 通配可以兜住全部，不用一个个列。

但通配会把 Gmail / Drive / Play 也拉进 Gemini 组，所以必须**按归属拆开 + 靠规则顺序控制**：

| 归属组 | 收什么 | 规则位置 |
|---|---|---|
| 📢 谷歌FCM | `fcm.googleapis.com`、`cloudmessaging-pa`、`firebaseinstallations(-pa)` | **第 1 位** |
| 🤖 OpenAi（gemini） | `growth-pa`、`notifications-pa`、`signaler-pa`、`feedback-pa`、`semanticlocation-pa`、`alkali*-pa` + 原 49 条具名 | **第 3 位** |
| 🤖 OpenAi（ai_all） | `appsgenaiserver-pa`、`cloudcode-pa`、`notebooklm-pa`、`antigravity-pa`、`geller-pa` … | **第 4 位** |
| Google | `pa.googleapis.com` 通配兜底（`accounts-pa` / `mail-pa` / `drive-pa` / `playstore-pa` …） | **第 6 位** |

**顺序是关键。** 原来的顺序是 `google`(3) → `gemini`(6)，`pa.googleapis.com` 通配放在 Google 组会把 AI 具名条目全抢走。改成：

```yaml
rules:
  - RULE-SET,google_fcm,📢 谷歌FCM
  - RULE-SET,tiktok,🎵 TikTok
  - RULE-SET,gemini,🤖 OpenAi       ← 具名优先
  - RULE-SET,ai_all,🤖 OpenAi
  - RULE-SET,muse,🤖 OpenAi
  - RULE-SET,google,Google          ← pa 通配兜底
  - RULE-SET,meta,📱 Meta
  - RULE-SET,taiwan_media,📺 台湾媒体
```

这样：AI 具名条目先命中 → 剩下的 `*-pa` 落到 Google 组 → 未来 Google 新增的 `-pa` 域名自动覆盖，不会再有漏网。

### 验证（`mihomo v1.19.14`，AI 组指 REJECT）

```
命中 🤖 OpenAi 13/13（000 = 被 REJECT）
  growth-pa  notifications-pa  signaler-pa  robinfrontend-pa
  feedback-pa  semanticlocation-pa  alkalicore-pa  aisandbox-pa
  gemini.google.com  bard.google.com  aistudio.google.com
  notebooklm.google.com  generativelanguage.googleapis.com

落到 Google / FCM 11/11（真实状态码 = DIRECT）
  accounts-pa 404   mail-pa 404   drive-pa 404   playstore-pa 404
  oauth2-pa 404     cloudmessaging-pa 404   firebaseinstallations 404
  play.google.com 302   gmail.com 301   www.google.com 200   x.com 200
```

provider 加载：`google_fcm 17 / gemini 58 / ai_all 213 / google 100`

### 顺带补的

`GoogleFCM.list` 原来只有 12 条 mtalk，**没有 `fcm.googleapis.com`**（截图里 `fcm.googleapis.com` 直连也是 404，但那是因为 Google 本来就不给根路径）。已补：

```
fcm.googleapis.com
cloudmessaging-pa.googleapis.com
firebaseinstallations-pa.googleapis.com
firebaseinstallations.googleapis.com
firebaseinstallations.googleapis.cn
```

---

## 🔗 rule-provider 为什么用 commit-sha 的 raw URL

### 两个坑

**坑 1：`@main` 的 raw URL 有 Fastly CDN 缓存**

实测：commit 后连续 6 次拉 `raw.githubusercontent.com/.../main/Clash/domain/Gemini-Full.yaml`，间隔 15 秒，共 90 秒，**全部返回旧内容**（1,926 B / 58 条），新加的 `signalr-pa` 一条都没有。

加 `?t=<时间戳>` 也无效 —— raw 的 CDN 对 query string 不做区分。

**换成 commit-sha 路径立刻就好：**

```
https://raw.githubusercontent.com/liuhui19994/ACL4SSR/main/...           → 1,926 B（旧）
https://raw.githubusercontent.com/liuhui19994/ACL4SSR/2d6961b/...        → 1,960 B（新，含 signalr-pa）
```

因为 sha 路径 CDN 从未缓存过，直接回源。

**坑 2：mihomo 按 `path` 做本地缓存**

`rule-providers` 的 `path:` 是本地落盘位置。如果只换 `url` 不换 `path`，mihomo 看到本地文件还在、`interval` 没到期，**就不重新下载**。

表现就是：改了 url、重启了 OpenClash，连接列表里规则还是老的。

**所以 `path` 必须跟着版本走：**

```yaml
gemini:
  type: http
  behavior: domain
  url: "https://raw.githubusercontent.com/liuhui19994/ACL4SSR/<sha>/Clash/domain/Gemini-Full.yaml"
  path: ./ruleset/gemini_v3.yaml      # ← 版本号写进文件名
  interval: 3600                       # ← 1 小时，不是 1 天
  proxy: DIRECT
```

### 更新规则时的完整流程

```bash
cd /root/.hermes/workspace/ACL4SSR-work
# 1. 改 Clash/domain/*.yaml
# 2. 提交
git add -A && git commit -m "..." && git push private main
# 3. 拿新 sha
SHA=$(git rev-parse HEAD)
# 4. 把两份 yaml 里 8 个 provider 的 url 换成新 sha，path 的版本号 +1
#    然后把新配置部署出去
```

**8 个走本仓库的 provider：**

| key | 文件 |
|---|---|
| `tiktok` | `Clash/domain/TikTok.yaml` |
| `google` | `Clash/domain/Google.yaml` |
| `meta` | `Clash/domain/Meta.yaml` |
| `muse` | `Clash/domain/Muse.yaml` |
| `gemini` | `Clash/domain/Gemini-Full.yaml` |
| `google_fcm` | `Clash/domain/GoogleFCM.yaml` |
| `ai_all` | `Clash/domain/AI-All.yaml` |
| `taiwan_media` | `Clash/domain/TaiwanMedia.yaml` |

另外 28 个 provider 仍指向 `cmliu/ACL4SSR` 上游（`@main` 和 `@433381eb` 两种），**没动**。
