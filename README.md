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
| `X.list` | 25 | x.com + twitter.com + twimg 图片/视频 CDN + t.co |
| `Muse.list` | 4 | `*.muse.ai` 全子域 + `meta.ai` |
| `Gemini-Full.list` | 49 | Gemini 网页 + App 私有端点 + NotebookLM + AI Studio |
| `GooglePlay-Full.list` | 21 | Play API + 图片 + **APK 下载 CDN gvt1/2/3** + 设备注册 |
| `GoogleFCM.list` | 12 | FCM 推送全部 mtalk 节点 |
| `AI-All.list` | 213 | ChatGPT / Claude / Grok / Perplexity / Copilot 等全量 |
| `TaiwanMedia.list` | 5 | LiTV / 4GTV / OFiii（台湾流媒体） |

> `Gemini-Full.list` 取代上游 `Gemini.list`（13 条，2025-06 停更）。
> `GooglePlay-Full.list` 取代上游 `PlayStore.list`（6 条）。
> 旧文件保留，不删，方便回退。

### `Clash/domain/*.yaml` — domain 格式（`behavior: domain`）

同样七份数据，换成 `payload:` 列表。**domain 行为比 classical 匹配更快，推荐用这个。**

### `yaml/clash-full.yaml` — 开箱即用配置

**在你现有 misub 配置上做加法，不是另起一套。** 分组、规则顺序、DNS 全部沿用你原来那份：

| 项目 | 原有 | 变动 | 现在 |
|---|---:|---|---:|
| 策略组 | 27 | +4（🎵 TikTok / 🐦 推特X / ▶️ 谷歌商店 / 📺 台湾媒体）<br>-8（奈飞 / 哔哩哔哩 / 微软×3 / 游戏平台 / 网易音乐 / 巴哈姆特） | **23** |
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
google-play       → GooglePlay-Full
googlefcm         → GoogleFCM
meta (muse 部分)  → Muse
```

`muse.ai` 的子域清单额外用 crt.sh 证书透明度日志核对过，共 29 个（`api` / `api1` / `api2` / `auth` / `files` / `live` / `vv` / `vv1-0-0` / `vv1-0-1` / `vv2-0-0` / `vv2-0-1` / `m00` / `m01` / `z` 等），`*.muse.ai` 一条通配全部覆盖。
