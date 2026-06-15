# miao-portfolio-workflow

A Claude skill for maintaining a personal portfolio website built with AI + deployed on Vercel.

**Target user**: PMs / designers / job seekers who built a static HTML portfolio with AI (Claude Code / Cursor / Gemini), deployed on Vercel, and want a reusable workflow for ongoing edits + deploys.

## What this skill does

Captures the full edit → sync → push → verify → deploy workflow:

- 🎨 **Edit source HTML/CSS/JS** in your local `personal-main/` directory
- 🔁 **Sync to a separate Vercel deploy repo** (the common "source vs deploy repo split" pattern)
- 🌐 **Commit + push via proxy** (WSL / mainland China network reliability)
- ✅ **Verify auto-build** via curl + grep
- 🧩 **Task templates** for 5 common changes: add project card, update copy, reorder sections, alternate dark/light backgrounds, add horizontal scroll nav buttons
- 🔗 **Bind custom domain** via Vercel REST API + DNS config (works for Aliyun, Tencent DNSPod, Cloudflare etc.)

## Install

```bash
# Clone into Claude's skills directory
git clone https://github.com/mengzheng001-prog/miao-portfolio-workflow.git \
  ~/.claude/skills/miao-portfolio-workflow
```

Or for project-scoped use:

```bash
git clone https://github.com/mengzheng001-prog/miao-portfolio-workflow.git \
  <your-project>/.claude/skills/miao-portfolio-workflow
```

Restart Claude. The skill auto-loads from the skills directory.

## When it triggers

This skill activates when you ask Claude to do any of:

- "在我的主页加一张项目卡 / 改副标题 / 换图标"
- "把工作经历调到核心项目前面"
- "给其他项目区加左右滑动按钮"
- "section 配色重新设计一下，深浅交替"
- "我买了个新域名 XXX，绑到 Vercel"
- "main.js / index.html 改完后怎么推上线"

Claude will read SKILL.md and walk through the 5-step workflow.

## Generalizable to different project setups

**v0.2 update**: SKILL.md now starts with a **Step 0 — Project Discovery** phase that auto-detects:

- Source directory location (`personal-main/` / `src/` / `site/` / `public/` / ...)
- Deploy mode (same repo vs separate deploy repo)
- CSS class naming (`bento-card` / `card` / Tailwind utilities / ...)
- Section IDs (`#projects` / `#experience` / your own naming)
- Network proxy needs (mainland China / WSL / VPN setups)
- DNS registrar (Aliyun / Tencent DNSPod / Cloudflare / Namecheap / GoDaddy)

So even if your project uses a different layout / CSS framework / DNS provider, the skill will adapt. The HTML snippets in `examples/` are reference points — Claude reads them, then applies your project's actual conventions.

Tested compatibility:

| Setup | Status |
|---|---|
| AI Studio / Vercel template (vanilla HTML) | ✅ Tested |
| Vite + vanilla HTML | ✅ Compatible |
| Tailwind utility-class projects | 🟡 Templates need light adapt |
| Next.js / SvelteKit | 🟡 Workflow applies, HTML templates don't |
| Webflow / Framer export | 🟡 Workflow applies, no HTML edits |

## What's in this repo

```
miao-portfolio-workflow/
├── SKILL.md                                    # Main skill spec (Claude reads this)
├── README.md                                   # This file
└── examples/
    ├── 01-add-personal-project-card.html       # HTML snippet for new project card
    └── 02-add-horizontal-nav-buttons.md        # frame + buttons + JS pattern
```

## Why this exists

When you build a portfolio with AI tools (Anthropic AI Studio, Vercel v0, Lovable), the typical setup is:

1. AI generates source files in one directory
2. You "deploy" by either uploading a zip to Vercel or pushing to a separate GitHub repo Vercel watches

The catch: **future edits split into "where you change code" vs "where Vercel reads code"**. Newcomers often only change source files and wonder why production doesn't update.

This skill encodes the round-trip workflow so Claude does it right every time — no more "I changed the file but it's not live".

## License

MIT — feel free to fork and adapt.

## Credits

Distilled from a real-world session maintaining [a personal portfolio](https://github.com/mengzheng001-prog/jia-yuya--) for an AI Product Manager job search, including project card additions, section reordering, dark/light background rebalancing, horizontal scroll nav, and custom domain binding via Aliyun DNS.
