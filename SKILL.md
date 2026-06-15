---
name: miao-portfolio-workflow
description: |
  维护一个用 AI 搭出来 + 部署在 Vercel（或类似平台）的静态作品集 / 简历主页的完整工作流。
  当用户要做这些事情时立刻调用：改个人主页 / 作品集页文案、加项目卡片（GitHub 链 / 桌面 app 链）、
  调整 section 顺序、调整 section 深浅背景配色交替、给横排卡片加左右滑动按钮、绑定新域名、
  把本地源文件同步到 Vercel 部署仓库 + push + 验证。
  即使用户没说"作品集 / 个人主页 / Vercel"等具体字眼，但提到"在我的简历网页 / 主页上加/改 XXX"
  也要主动用这个 skill。skill 本身设计为通用的——首次跑时会先探查项目结构、CSS 约定、网络环境，
  再 adapt 到具体项目。
---

# Maintaining a Vercel-deployed personal portfolio site

适用对象：用 AI 搭出过一份静态 HTML 作品集（含项目卡片、section 分块）、部署在 Vercel / Netlify / Cloudflare Pages、绑定了自定义域名，需要长期增删项目卡 / 改文案 / 调结构。

## Why this skill exists

这类项目通常有个**本地源文件 vs 部署仓库分离**的特点：
- 用户在**本地源目录**改 HTML / CSS / JS
- 平台（Vercel/Netlify/...）监听的是**另一个 GitHub 仓库**，改完要手动同步过去
- 部分地区开发者直连 GitHub.com 和 *.vercel.app 经常被网络劫持，需要走代理

新手最容易踩的坑：只改了源文件没同步部署 repo → 推线上的是旧版 → 验证看不到改动 → 怀疑代码出错。

这个 skill 让 Claude 第一次就走完整条链路：**源改 → 同步 → commit → push → 验证**。

---

## Step 0 — Project Discovery（首次使用时跑一遍）

⚠️ **第一次给某个项目用这个 skill 时，先花 2 分钟探查项目结构**。不要假设别人的项目跟示例完全一致。把探查结果**告诉用户并写到对话上下文里**，后续步骤用这些变量替代占位符。

### 0.1 找源目录

```bash
ls <PROJECT_ROOT>/
# 看有没有这些常见名字：
#   personal-main/ / src/ / site/ / public/ / pages/
# 或者 .html 直接在根目录（vanilla 项目）
find <PROJECT_ROOT> -maxdepth 2 -name "index.html" -not -path "*/node_modules/*" 2>/dev/null
```

记下 `<SOURCE_DIR>` = 实际源目录路径。

### 0.2 看部署 repo 是分离的还是同一个

```bash
cd <PROJECT_ROOT>
git remote -v 2>/dev/null
ls -la .vercel/ 2>/dev/null    # Vercel CLI link 文件
cat vercel.json 2>/dev/null    # Vercel 配置（如有）
```

- 有 `.git` 且 remote 指向某个 GitHub repo → **同一个 repo** 直接改源即可，push 这个 repo 就部署
- 没有 `.git` 或 remote 指向另一个仓库 → **分离模式**，需要 sync 到 `<DEPLOY_REPO>`

记下 `<DEPLOY_MODE>` = `same` 或 `separate`，记下 `<DEPLOY_REPO>` URL（如果分离）。

### 0.3 探查 CSS 类命名约定

```bash
# 看横排卡片容器的命名（grid / row / cards 等）
grep -oE 'class="[^"]*grid[^"]*"' <SOURCE_DIR>/index.html | sort -u | head -10

# 看卡片本身命名（card / item / project 等）
grep -oE 'class="[^"]*card[^"]*"' <SOURCE_DIR>/index.html | sort -u | head -10

# 看 section 命名 + 深浅 class
grep -oE '<section[^>]*>' <SOURCE_DIR>/index.html | head -10
```

记下：
- `<GRID_CLASS>` = 横排卡片容器类名（如 `other-grid` / `projects-grid` / `tw-flex`）
- `<CARD_CLASS>` = 单张卡片类名（如 `bento-card other-card` / `project-card`）
- `<ALT_CLASS>` = 深色 section 切换类名（如 `alt` / `dark` / `bg-slate-50`），可能不存在
- `<SECTION_IDS>` = 现有 section 列表（如 `#projects`, `#experience`, ...）

### 0.4 看构建工具

```bash
cat <PROJECT_ROOT>/package.json 2>/dev/null | python3 -c "
import sys, json
try:
  d = json.load(sys.stdin)
  print('scripts:', list(d.get('scripts',{}).keys()))
  print('deps:', list(d.get('dependencies',{}).keys())[:5])
except: pass
"
```

- 有 `vite` → Vite 项目，输出在 `dist/`
- 有 `next` → Next.js，输出在 `.next/` 或 `out/`
- 没 package.json → vanilla 静态站，直接 serve 源文件

记下 `<BUILD_TOOL>`。如果是 vanilla 站，部署 repo 直接 cp HTML/CSS/JS；如果是构建型项目，可能要先 `npm run build` 把产物 push。

### 0.5 探查网络是否需要代理

```bash
# 试直连
curl -s --max-time 8 -o /dev/null -w "%{http_code}\n" https://github.com/ 2>&1
curl -s --max-time 8 -o /dev/null -w "%{http_code}\n" https://api.vercel.com/v2/user 2>&1

# 看 env 是否设了代理
env | grep -iE "^(http|https|all)_proxy"
```

- 直连两条都返回 `200` / `401` → **不需要代理**，所有 git 操作直连即可
- 直连超时或 timeout → **需要代理**，找出端口（Clash 默认 7890，ClashX 10090，SS 1080，Verge mix-port 7897 等）

记下 `<PROXY_CMD>`：
- 不需要 → 空字符串
- 需要 → `export https_proxy=http://127.0.0.1:<PORT> http_proxy=http://127.0.0.1:<PORT>`

### 0.6 探查 DNS 注册商（仅绑域名时需要）

如果当前任务是绑定新域名，问用户「域名是在哪买的」：
- 阿里云万网 → `dns.console.aliyun.com`
- 腾讯云 DNSPod → `console.dnspod.cn`
- Cloudflare → `dash.cloudflare.com/domains`
- Namecheap / GoDaddy / Spaceship → 各自后台

不同注册商「加 A 记录 + CNAME」的具体路径不同，但记录内容（IP / CNAME 值）是 Vercel 统一给的，**这部分不变**。

### 0.7 总结探查结果给用户

把上面的发现按这个模板告诉用户：

```
📋 项目探查完成：
  源目录：       <SOURCE_DIR>
  部署模式：     <DEPLOY_MODE>（同 repo / 分离）
  部署 repo：    <DEPLOY_REPO>
  构建工具：     <BUILD_TOOL>
  CSS grid 类：  <GRID_CLASS>
  卡片类：       <CARD_CLASS>
  section ID：   <SECTION_IDS>
  深浅 class：   <ALT_CLASS>
  代理需求：     <PROXY_CMD or "直连即可">
  域名注册商：   <REGISTRAR>（如适用）

后续 task templates 会用你项目的实际类名（不是示例里的）。
```

✅ **从这里开始，下文里所有 `<占位符>` 都替换成上面探查到的实际值**。如果项目结构跟下面示例完全一致，沿用即可；不一致时按你项目的命名做。

---

## Standard workflow（任何改动都走这 5 步）

> **下面的命令用 Step 0 探查到的实际变量替换**。如果你的项目 `<DEPLOY_MODE> = same`（同 repo），跳过 Step 2 的 cp 操作，直接在源目录 commit + push。

### Step 1 — 改源文件

只改 `<SOURCE_DIR>/` 下的文件（Step 0.1 探查到的源目录）。**不要直接改部署 repo 的副本** —— 部署 repo 只是「同步目标」，保持单向流（源 → 部署）。

### Step 2 — Sync 到部署 repo（仅 `<DEPLOY_MODE> = separate` 时执行）

如果 `/tmp/<DEPLOY_REPO_NAME>/` 还没 clone，第一次先 clone：

```bash
<PROXY_CMD>      # Step 0.5 探查到的代理命令；不需要代理时这行不写
git clone <DEPLOY_REPO_URL> /tmp/<DEPLOY_REPO_NAME>
```

之后每次 push 前 pull 一下：

```bash
cd /tmp/<DEPLOY_REPO_NAME> && git pull origin main
```

用 `cp` 把改动的文件覆盖过去（**只 cp 改了的**，不要 `cp -r .`，避免把 .git 和无关文件覆盖掉）：

```bash
cp <SOURCE_DIR>/index.html        /tmp/<DEPLOY_REPO_NAME>/index.html
cp <SOURCE_DIR>/assets/main.js    /tmp/<DEPLOY_REPO_NAME>/assets/main.js
cp <SOURCE_DIR>/assets/style.css  /tmp/<DEPLOY_REPO_NAME>/assets/style.css
```

> 如果项目是 Vite/Next 等**构建型**（Step 0.4 `<BUILD_TOOL>` 不是 vanilla）：先在源目录 `npm run build`，然后 cp `dist/` 或 `out/` 整体到部署 repo，而不是 cp 单文件。

### Step 3 — Commit

```bash
cd /tmp/<DEPLOY_REPO_NAME>     # 或 <PROJECT_ROOT>（如果 DEPLOY_MODE = same）
git -c user.email="<EMAIL>" -c user.name="<USERNAME>" \
    commit -am "<conventional commit message>"
```

Commit message 建议格式：
- `feat: 加 XXX 项目卡` —— 新功能 / 新内容
- `tweak: <section> 副标题改为 YYY` —— 小幅文案 / 配色调整
- `fix: 工作经历 section 实际位置也提到核心项目之前` —— 修 bug
- `style: section 深浅奇偶交替` —— 纯样式

### Step 4 — Push

```bash
<PROXY_CMD>      # Step 0.5 探查到的；不需要代理时这行不写
git push origin main
```

### Step 5 — 验证

Vercel 监听到 push 后自动 build，**通常 30 秒-1 分钟内生效**。验证方法：

```bash
sleep 30
<PROXY_CMD>      # 只在需要代理时加

# HTTP 状态
curl -s --max-time 15 -o /dev/null -w "HTTP %{http_code}\n" "https://<LIVE_URL>/"

# 关键字检查（grep 你刚改的关键字，命中说明新版本生效）
curl -s --max-time 15 "https://<LIVE_URL>/" | grep -c "<刚加的关键字>"
```

`HTTP 200` + 关键字命中 ≥ 1 → 主线完成。

> `<LIVE_URL>` 优先用绑定的自定义域名；没绑域名时用 `<USERNAME>.vercel.app`。部分代理会拦 *.vercel.app，可切自定义域名验证。

## Task templates（5 种典型场景）

> ⚠️ **下面的 HTML / CSS 类名、section ID 是「同款 AI Studio 模板」的实例**——给你直观参考。
> 你项目里实际用什么，请用 Step 0.3 探查到的 `<GRID_CLASS>` / `<CARD_CLASS>` / `<SECTION_IDS>` 替换。
> 如果项目结构完全不同（如 Tailwind utility class、React component），只借**思路**，不用照搬 HTML。

### Template A · 加一张项目卡

作品集里通常有 2-3 块项目区，每块定位不同：

- **核心 / 代表项目**：用户主导完成的，每张卡详细 KV，**点击进子页**
- **其他项目**：参与过的项目模块，横排滑动卡片
- **个人项目 / 开源**：独立开发的小工具，**点击直跳 GitHub repo**

加卡时**复制同区现有卡的 HTML 结构**（grep 同区第一张卡整段复制），改 5 个东西：

1. `href` 指向 GitHub repo / 子页
2. 顶部 tag（一句技术栈）
3. `<h3>` 标题
4. 描述段（80-130 字定位 + 核心做法）
5. KV / metadata 行（技术栈、解决痛点、性能数据）

完整模板见 `examples/01-add-personal-project-card.html`（同款 AI Studio 模板示例；不同框架的项目按思路改）。

### Template B · 改 section 副标题 / 标题

**先 grep 定位**，避免改错位置（同文案可能在多处出现）：

```bash
grep -n "<旧文案>" <SOURCE_DIR>/index.html
```

如果同一段文案在多处（导航栏 + footer + hero），**逐个 Edit** 或用 `replace_all`。

### Template C · 调整 section 顺序

`<section id="X">` 整块剪切粘贴。每个 section 边界很清晰：`<section id=...>` 到对应 `</section>`。

通用步骤：
1. 读要挪动的 section 完整内容（一般 60-150 行）
2. 把它**插入到目标位置前**（Edit 目标 section 起始行，前面拼上要挪的整块）
3. 把**原来位置的 section 整块删掉**（Edit 替换为空）

⚠️ **完成后顺手核对 3 个地方是否对应了**：
- 导航栏锚点链接（`<a href="#X">`）的视觉顺序
- Section 渲染的视觉顺序
- 任何 hero 区 CTA 按钮指向（`<a href="#X">查看 XXX</a>`）

3 处不一致会让用户疑惑。

### Template D · 深浅背景奇偶交替

很多模板用一个 modifier class（示例里叫 `alt`，其它项目可能叫 `dark` / `bg-muted` / `tw-bg-slate-50` 等，参考 Step 0.3 探查到的 `<ALT_CLASS>`）来切换 section 深浅背景。

调整 section 顺序后，可能出现「连续两块同色」破坏分段感。**修法：toggle 这个 modifier class，让深浅严格交替**。

```html
<!-- Before：改完顺序后 #A + #B 都是 alt 深色 → 连两块 -->
<section id="A" class="alt section-reveal">
<section id="B" class="section-reveal">

<!-- After：A 改浅，B 改深 -->
<section id="A" class="section-reveal">
<section id="B" class="alt section-reveal">
```

按现有 section 序列规划一个 1/3/5/7… 奇数为深、偶数为浅（或反过来）的方案，整体 toggle 一遍。

> 如果项目用 Tailwind，深色可能是 `class="bg-slate-50"`，浅色是无 class。同样的原则——toggle 一个 utility class 即可。

### Template E · 加横向滑动按钮（左右箭头）

横排卡片区原本只能用鼠标滚轮横滑，**加左右圆形箭头按钮**体验更好。

完整 HTML 结构 + JS 见 `examples/02-add-horizontal-nav-buttons.md`。

核心要点（**用你项目的 `<GRID_CLASS>` 替换示例里的 `other-grid`**）：
1. 把现有 `<div class="<GRID_CLASS>">` 包一层 `<div class="<GRID_CLASS>-frame">`（命名按你项目约定），frame 内加两个 `<button class="grid-nav grid-nav-prev/next">`
2. **JS 必须用 `querySelectorAll('.<GRID_CLASS>-frame').forEach()`** 遍历所有 frame —— 否则只绑第一个区，第二个区的按钮点不动。这个坑很容易犯（如果项目原本只有一个横排区，原 JS 用 `querySelector` 单数取 grid，加第二个区时必须改为复数 + forEach）

## Vercel 域名绑定（高级）

这步通常只做一次。前提：用户已经在 Vercel 后台关联了 GitHub 部署 repo + 网站能用 `<USERNAME>.vercel.app` 访问。

### 路径选择

| 方式 | 适合 |
|---|---|
| **Vercel CLI**（`npm i -g vercel && vercel login` 浏览器授权） | 长期维护，token 不离开本机 |
| **REST API + 临时 token** | 单次绑定，最快，5 分钟搞定 |
| **手动后台**（Vercel + 注册商后台） | 不愿装 CLI / 给 token 的非技术用户 |

用 REST API 方式时，让用户：

1. 去 https://vercel.com/account/tokens 创建一个 **scoped token**（scope 选具体项目，expiration 选 1 day）
2. **不要在对话里贴 token** —— 写到 `/tmp/vc.token` 文件，`chmod 600`
3. 完成后立即 `rm /tmp/vc.token` + 让用户去后台 Revoke

API 调用（绑两个域名：apex + www）：

```bash
export VC_TOKEN=$(cat /tmp/vc.token)
unset HTTPS_PROXY HTTP_PROXY https_proxy http_proxy

# 1. 验证 token + 取 team_id（Hobby plan 也有 team）
curl -s -H "Authorization: Bearer $VC_TOKEN" "https://api.vercel.com/v2/teams" | python3 -m json.tool

# 2. 加 apex 域名
curl -s -X POST -H "Authorization: Bearer $VC_TOKEN" -H "Content-Type: application/json" \
  "https://api.vercel.com/v10/projects/<PROJECT_ID>/domains?teamId=<TEAM_ID>" \
  -d '{"name":"<YOUR_DOMAIN>"}'

# 3. 加 www 子域
curl -s -X POST -H "Authorization: Bearer $VC_TOKEN" -H "Content-Type: application/json" \
  "https://api.vercel.com/v10/projects/<PROJECT_ID>/domains?teamId=<TEAM_ID>" \
  -d '{"name":"www.<YOUR_DOMAIN>"}'

# 4. 查 DNS 配置要求（关键！Vercel 给的 IP / CNAME 时常变）
curl -s -H "Authorization: Bearer $VC_TOKEN" \
  "https://api.vercel.com/v6/domains/<YOUR_DOMAIN>/config?teamId=<TEAM_ID>" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('A:', d.get('recommendedIPv4')); print('CNAME:', d.get('recommendedCNAME'))"
```

### 在域名注册商后台加 DNS 解析

用户**自己在注册商后台加**——你没有 DNS 权限。不同注册商位置不一样，但**记录内容来自上一步 Vercel API 返回的值**：

| 主机记录 | 类型 | 记录值（**以 Vercel API 实际返回的为准**） | TTL |
|---|---|---|---|
| `@` (apex) | A | `<Vercel 推荐 IP，常见 216.198.79.1 / 76.76.21.21>` | 600 秒（10 分钟） |
| `www` | CNAME | `<project-hash>.vercel-dns-017.com`（末尾不带点）或 `cname.vercel-dns.com` | 600 秒 |

各注册商后台路径速查：

| 注册商 | 路径 |
|---|---|
| **阿里云万网** | dns.console.aliyun.com → 域名解析 → 选域名 → 添加记录；负载策略选「轮询」 |
| **腾讯云 DNSPod** | console.dnspod.cn → 我的域名 → 选域名 → 解析 → 添加记录 |
| **Cloudflare** | dash.cloudflare.com → 选域名 → DNS → Records → Add record；**Proxy status 选 "DNS only"**（橙色云朵关掉，否则跟 Vercel SSL 冲突） |
| **Namecheap** | ap.www.namecheap.com → Domain List → Manage → Advanced DNS → Add New Record |
| **GoDaddy** | dcc.godaddy.com → My Products → 选域名 → DNS → Add → Type A/CNAME |
| **Spaceship** | spaceship.com → My Domains → Manage → DNS Records → Add |

> Cloudflare 是个特殊情况：默认会把流量代理到 Cloudflare CDN（橙色云图标）。如果保持代理打开 + 用 Vercel，会导致 SSL 证书冲突。**必须把 Proxy status 切到 "DNS only"（灰色云图标）**。

### 验证

```bash
# 1. DNS 配置 OK（Vercel API 不再 misconfigured）
curl -s -H "Authorization: Bearer $VC_TOKEN" \
  "https://api.vercel.com/v6/domains/<YOUR_DOMAIN>/config?teamId=<TEAM_ID>" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('misconfigured:', d.get('misconfigured'))"

# 2. HTTPS 能访问（SSL 由 Vercel 自动签 Let's Encrypt）
<PROXY_CMD>      # 只在需要代理时加
curl -s --max-time 20 -o /dev/null -w "HTTP %{http_code}\n" "https://<YOUR_DOMAIN>/"
```

✅ `misconfigured: False` + `HTTP 200` → 域名绑定完成。

完事后立即 `rm /tmp/vc.token` + 让用户去 Vercel Tokens 页 Revoke。

## Common pitfalls（避坑）

| 现象 | 原因 | 解法 |
|---|---|---|
| 改了 `personal-main/` 但 Vercel 没变 | 忘了 sync 到部署 repo | 跑 Step 2-4 |
| `git push` 卡住 145 秒后 timeout | WSL 直连 github.com 被劫持 | `export https_proxy=http://127.0.0.1:<PORT>` |
| 加了导航锚点但 section 顺序没动 | 锚点 ≠ section 实际位置 | Template C 整块剪切 |
| 两块相邻 section 同色背景 | 共用 `<ALT_CLASS>` | Template D toggle |
| 第二个横滑区按钮点不动 | JS 用 `querySelector` 单数只绑了第一个 | 改为 `querySelectorAll('.<GRID_CLASS>-frame').forEach` |
| 新域名 `HTTP 200` 但显示「This deployment can not be found」 | 域名加到了错误的 project | 检查 `<PROJECT_ID>` 是否正确 |
| Vercel API 报 `400 The team does not have ...` | 漏了 `?teamId=...` 参数 | Hobby plan 也有 team_id，必须带 |
| Cloudflare 域名绑了但页面卡 SSL 错误 | Cloudflare Proxy 没关 | DNS Records 把 Proxy status 切到「DNS only」（灰云） |
| 跨项目用同 skill 时步骤"想当然"出错 | 跳过了 Step 0 项目探查 | 第一次给某项目用时，必须先跑 Step 0 |

## Sanity checklist（每次改完跑一遍）

- [ ] 源文件 `<SOURCE_DIR>/` 改完了
- [ ] （如 `<DEPLOY_MODE>` = separate）sync 到 `/tmp/<DEPLOY_REPO_NAME>/`
- [ ] `git status --short` 只显示我改的文件，**没有额外文件**
- [ ] commit message 符合 `feat / tweak / fix / style` 模板
- [ ] `git push` 成功（看到 "→ main" 那行）
- [ ] sleep 30 后 `curl https://<LIVE_URL>/` 返回 HTTP 200
- [ ] `grep -c "<新加关键字>"` 命中 ≥ 1
- [ ] （绑域名后）`misconfigured: False`、SSL 有效

走完这 8 步 → 真的改完了，告诉用户「已上线 + 给链接」。

---

## Skill 自检：要不要跳过 Step 0？

跳过 Step 0 的合理情况：
- ✅ 同一个项目你已经做过几次改动，结构都记着了
- ✅ 用户明说「我项目跟示例完全一样」

**不能**跳过 Step 0 的情况（即使用户催你快）：
- ❌ 第一次给某个新项目用这个 skill
- ❌ 用户给了一个新的工作目录路径
- ❌ 用户提到「我朋友/我同事的简历主页」（不是你之前服务过的那个项目）

错过 Step 0 的成本是：套了一堆错的类名 / 错的目录 → push 一堆错误改动 → 用户疑惑「为什么页面没变 / 变错了」。**Step 0 多花 2 分钟，比 reverse 一次错误 commit 便宜得多**。
