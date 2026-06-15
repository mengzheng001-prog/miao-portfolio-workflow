---
name: miao-portfolio-workflow
description: |
  维护一个用 AI 搭出来 + 部署在 Vercel 的静态作品集 / 简历主页的完整工作流。
  当用户要做这些事情时立刻调用：改个人主页 / 作品集页文案、加项目卡片（GitHub 链 / 桌面 app 链）、
  调整 section 顺序、调整 section 深浅背景配色交替、给横排卡片加左右滑动按钮、绑定新域名（Vercel + 阿里云 DNS）、
  把本地源文件同步到 Vercel 部署仓库 + push + 验证。
  即使用户没说"作品集 / 个人主页 / Vercel"等具体字眼，但提到"在我的简历网页 / 主页上加/改 XXX"
  也要主动用这个 skill。
---

# Maintaining a Vercel-deployed personal portfolio site

适用对象：产品经理 / 设计师 / 求职者，用 AI 搭出过一份静态 HTML 作品集（含项目卡片、工作经历、section 分块）、部署在 Vercel、绑定了自定义域名，需要长期增删项目卡 / 改文案 / 调结构。

## Why this skill exists

这类项目有个**本地源文件 vs Vercel 部署仓库分离**的特点：
- 用户在**本地源目录**（如 `personal-main/`）改 HTML / CSS / JS
- Vercel 监听的是**另一个独立 GitHub 仓库**（如 `<DEPLOY_REPO>`），改完要手动同步过去
- 中国大陆开发者在 WSL / 终端里直连 GitHub.com 和 vercel.app 经常被网络劫持，需要走 socks/http proxy

新手很容易踩的坑：只改了源文件没同步部署 repo → 推 Vercel 的是旧版 → 验证看不到改动 → 怀疑代码出错。

这个 skill 让 Claude 第一次就走完整条链路：**源改 → 同步 → commit → push → 验证**。

## Project layout（你需要先了解的）

```
<PROJECT_ROOT>/                              # 用户 AI Studio / Vercel 拉下的本地源
├── personal-main/                           # 源目录（你改的位置）
│   ├── index.html                           # 主页 HTML
│   ├── archiai.html / tourism.html / ...    # 项目子页（可能有多个）
│   └── assets/
│       ├── style.css
│       └── main.js                          # 交互逻辑（横滑、灯箱等）
```

部署链路（独立的 GitHub repo + Vercel 监听）：

```
GitHub: <USERNAME>/<DEPLOY_REPO>             # Vercel "Import Git Repository" 关联的仓库
   ↓ Vercel 监听 main 分支 push 自动 build
Vercel: <USERNAME>.vercel.app                 # 默认 Vercel 域名
   ↓ 后台 Domains 加自定义域名
Custom: <YOUR_DOMAIN>                         # 用户买的域名
```

**本地 clone 部署仓库**到一个临时位置（不放工作目录，避免跟源混淆）：
```
/tmp/<DEPLOY_REPO>/                          # 改完用来 push 的"中转站"
```

## Standard workflow（任何改动都走这 5 步）

### Step 1 — 改源文件

只改 `personal-main/` 下的文件。**不要直接改部署 repo 的副本** —— 部署 repo 只是「同步目标」，保持单向流（源 → 部署）。

### Step 2 — Sync 到部署 repo

如果 `/tmp/<DEPLOY_REPO>` 还没 clone，第一次先 clone（走 proxy，原因见下文）：

```bash
unset HTTPS_PROXY HTTP_PROXY https_proxy http_proxy
git clone https://github.com/<USERNAME>/<DEPLOY_REPO>.git /tmp/<DEPLOY_REPO>
```

之后每次 push 前 pull 一下保持同步：

```bash
cd /tmp/<DEPLOY_REPO> && git pull origin main
```

然后用 `cp` 把改动的文件覆盖过去（只 cp 改了的，不要 `cp -r .`，否则会把无关文件也带过去）：

```bash
cp <PROJECT_ROOT>/personal-main/index.html        /tmp/<DEPLOY_REPO>/index.html
cp <PROJECT_ROOT>/personal-main/assets/main.js    /tmp/<DEPLOY_REPO>/assets/main.js
cp <PROJECT_ROOT>/personal-main/assets/style.css  /tmp/<DEPLOY_REPO>/assets/style.css
```

### Step 3 — Commit

在 `/tmp/<DEPLOY_REPO>` 内 commit。第一次 commit 要带 user.email / user.name（如果机器全局没设过）：

```bash
cd /tmp/<DEPLOY_REPO>
git -c user.email="<EMAIL>" -c user.name="<USERNAME>" \
    commit -am "<conventional commit message>"
```

Commit message 建议格式：
- `feat: 加 XXX 项目卡` —— 新功能 / 新内容
- `tweak: <section> 副标题改为 YYY` —— 小幅文案 / 配色调整
- `fix: 工作经历 section 实际位置也提到核心项目之前` —— 修 bug
- `style: section 深浅奇偶交替（experience 改浅 / projects 改深）` —— 纯样式

### Step 4 — Push（走 proxy）

WSL / Linux 直连 GitHub.com 经常出现 `Failed to connect to github.com port 443 after 145734 ms` 或 `GnuTLS recv error`。**走 proxy 一次就过**：

```bash
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890
git push origin main
```

> Proxy 端口因人而异（7890 是 Clash 默认，10090 是 ClashX，1080 是 SS）。如果用户机器上没在跑代理 → 让用户开了再试。

### Step 5 — 验证

Vercel 监听到 push 后自动 build，**通常 30 秒-1 分钟内生效**。验证方法：

```bash
sleep 30
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890

# HTTP 状态
curl -s --max-time 15 -o /dev/null -w "HTTP %{http_code}\n" "https://<YOUR_DOMAIN>/"

# 关键字检查（grep 你刚改的关键字，命中说明新版本生效）
curl -s --max-time 15 "https://<YOUR_DOMAIN>/" | grep -c "<新加的关键字>"
```

`HTTP 200` + 关键字命中 ≥ 1 → 主线完成。

## Task templates（4 种典型场景）

### Template A · 加一张项目卡

作品集里通常有「核心项目」「其他项目」「个人项目」3 块项目区。新加项目时确定放哪一块：

- **核心项目**（`#projects`）：用户主导完成的代表性项目，每张卡通常带 metrics / 详细 KV，**点击进子页**（`archiai.html`、`tourism.html` 等）
- **其他项目**（`#other-projects`）：参与过的项目模块，横排滑动卡片
- **个人项目**（`#personal-projects`）：独立开发的开源小工具，**点击直跳 GitHub repo**

加卡时**复制同区现有卡的 HTML 结构**，改 5 个东西：
1. `href` 指向 GitHub repo / 子页
2. 顶部 tag（如 `Python · 桌面工具 · 音频`）
3. `<h3>` 标题
4. `<p class="desc">` 一段 80-130 字的描述
5. `kv-row` 里的「技术栈」「解决痛点」（kv-row 数量按需）

完整模板见 `examples/01-add-personal-project-card.html`。

### Template B · 改 section 副标题 / 标题

通常是 `<h2 class="section-title">` 和 `<p class="section-subtitle">` 两行：

```bash
grep -n "<旧文案>" <PROJECT_ROOT>/personal-main/index.html
```

定位后用 Edit 单点替换。注意：如果同一段文案在多页都有（如导航栏链接），用 `replace_all` 或者逐个改。

### Template C · 调整 section 顺序

`<section id="X">` 整块剪切粘贴。每个 section 边界很清晰：`<section id=...>` 到对应 `</section>`。

例：把「工作经历」（`#experience`）移到「核心项目」（`#projects`）前面 →
1. 读 `#experience` 整块（一般 60-100 行）
2. 把它**插入到 `<!-- Projects -->` 注释前**（Edit `<!-- Projects -->` 行，前面拼上 experience 块）
3. 把原来位置的 experience 块**整块删掉**（Edit 把 `<!-- Experience -->...</section>\n\n    <!-- Education -->` 替换成 `<!-- Education -->`）

**完成后顺手核对**：导航栏锚点链接的视觉顺序、跟 section 视觉顺序是不是匹配（容易漏 sync）。

### Template D · 深浅背景奇偶交替

`section.alt` 在 CSS 里是带 30% 半透明深色背景的 section。section 顺序调整后，可能出现「连续两块同色」破坏分段感。

修法：toggle 单字符 `alt` —— 让深浅严格交替即可。

```html
<!-- Before: 改完顺序后 experience + projects 都是 alt 深色 → 连两块 -->
<section id="experience" class="alt section-reveal">
<section id="projects" class="section-reveal">

<!-- After: experience 改浅，projects 改深 -->
<section id="experience" class="section-reveal">
<section id="projects" class="alt section-reveal">
```

按现有 section 序列，规划一个 1/3/5/7… 奇数为深、偶数为浅（或反过来）的方案，整体 toggle 一遍。

### Template E · 加横向滑动按钮（左右箭头）

「其他项目」「个人项目」这种横排卡片区，原本只能用鼠标滚轮横滑，**加左右圆形箭头按钮**体验更好。

完整 HTML 结构 + JS 见 `examples/02-add-horizontal-nav-buttons.md`。

核心要点：
1. 把现有 `<div class="other-grid">` 包一层 `<div class="other-grid-frame">`，frame 内加两个 `<button class="grid-nav grid-nav-prev/next">`
2. **JS 必须用 `querySelectorAll('.other-grid-frame').forEach()`** 遍历所有 frame —— 否则只绑第一个区，第二个区的按钮点不动

## Vercel 域名绑定（高级）

这步通常只做一次。前提：用户已经在 Vercel 后台关联了 GitHub 部署 repo + 网站能用 `<USERNAME>.vercel.app` 访问。

### 路径选择

| 方式 | 适合 |
|---|---|
| **Vercel CLI**（`npm i -g vercel && vercel login` 浏览器授权） | 长期维护，token 不离开本机 |
| **REST API + 临时 token** | 单次绑定，最快，5 分钟搞定 |
| **手动后台 + Aliyun 后台** | 不愿装 CLI / 给 token 的非技术用户 |

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

### 阿里云万网（或腾讯云 DNSPod）加解析

用户**自己在阿里云后台加**——你没有 DNS 权限。给清晰的指引：

| 主机记录 | 类型 | 记录值（**以上一步 Vercel API 返回的为准**） | TTL |
|---|---|---|---|
| `@` | A | `<Vercel 推荐 IP，如 216.198.79.1>` | 10 分钟 |
| `www` | CNAME | `<project-hash>.vercel-dns-017.com`（末尾不带点） | 10 分钟 |

负载策略选「**轮询**」（即使只填一个 IP 也选轮询，方便以后加备用 IP）。

### 验证

```bash
# 1. DNS 配置 OK（Vercel API 不再 misconfigured）
curl -s -H "Authorization: Bearer $VC_TOKEN" \
  "https://api.vercel.com/v6/domains/<YOUR_DOMAIN>/config?teamId=<TEAM_ID>" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('misconfigured:', d.get('misconfigured'))"

# 2. HTTPS 能访问（SSL 由 Vercel 自动签 Let's Encrypt）
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890
curl -s --max-time 20 -o /dev/null -w "HTTP %{http_code}\n" "https://<YOUR_DOMAIN>/"
```

✅ `misconfigured: False` + `HTTP 200` → 域名绑定完成。

完事后立即 `rm /tmp/vc.token` + 让用户去 Vercel Tokens 页 Revoke。

## Common pitfalls（避坑）

| 现象 | 原因 | 解法 |
|---|---|---|
| 改了 `personal-main/` 但 Vercel 没变 | 忘了 sync 到部署 repo | 跑 Step 2-4 |
| `git push` 卡住 145 秒后 timeout | WSL 直连 github.com 被劫持 | `export https_proxy=http://127.0.0.1:7890` |
| 加了导航锚点但 section 顺序没动 | 锚点 ≠ section 实际位置 | Template C 整块剪切 |
| 两块相邻 section 同色背景 | 共用 `class="alt"` | Template D toggle |
| 第二个横滑区按钮点不动 | JS 只绑了 `querySelector('.other-grid')` 单数 | 改为 `querySelectorAll('.other-grid-frame').forEach` |
| 新域名 `HTTP 200` 但显示「This deployment can not be found」 | 域名加到了错误的 project | 检查 `projectId` 是否正确 |
| Vercel API 报 `400 The team does not have ...` | 漏了 `?teamId=...` 参数 | Hobby plan 也是有 team_id 的，必须带 |

## Sanity checklist（每次改完跑一遍）

- [ ] 源文件 `personal-main/` 改完了
- [ ] sync 到 `/tmp/<DEPLOY_REPO>/`
- [ ] `git status --short` 只显示我改的文件，**没有额外文件**
- [ ] commit message 符合 `feat / tweak / fix / style` 模板
- [ ] `git push` 成功（HTTP 200 / "→ main" 行）
- [ ] sleep 30 后 `curl https://<YOUR_DOMAIN>/` 返回 HTTP 200
- [ ] `grep -c "<新加关键字>"` 命中 ≥ 1
- [ ] （绑域名后）`misconfigured: False`、`SSL` 有效

走完这 8 步 → 真的改完了，告诉用户「已上线 + 给链接」。
