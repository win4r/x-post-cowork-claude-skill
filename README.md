# x-post-cowork-claude-skill

Publish posts to **X (Twitter)** via **Claude Cowork / Claude Code** browser automation using **Claude in Chrome** tools.

This is a **Cowork-native adaptation** of [win4r/x-post-browser-skill](https://github.com/win4r/x-post-browser-skill) (originally built for OpenClaw). It uses Claude in Chrome MCP tools instead of OpenClaw's browser API.

## What it does

- Opens X (x.com) in your Chrome browser
- Opens the **compose dialog**
- Inserts your post text via DOM injection (reliable, no hashtag corruption)
- Clicks **Post**
- Verifies success and returns the permalink

## Two input modes

### 1. Topic mode (draft + confirm)

You provide a topic/requirements, Claude writes a Chinese post (~100 chars) with 3–6 English hashtags, sends you the draft, and only posts after you reply **"发布"/"发"**.

### 2. Raw-text mode (publish immediately)

You provide the final post text (prefixed with `发X：`, `tweet:`, etc.), Claude posts it **verbatim** without confirmation.

## Requirements

- **Claude Cowork** (desktop app) or **Claude Code** with Claude in Chrome extension
- **Chrome browser** with the Claude in Chrome extension installed and connected
- **Logged in to X** in that Chrome browser profile

## Installation

### Option A: Install the .skill file

Download the `.skill` file from [Releases](https://github.com/win4r/x-post-cowork-claude-skill/releases), then install it in Cowork.

### Option B: Manual install

Copy `SKILL.md` into your Cowork skills directory.

## Usage examples

### Topic mode

```
发布一条 X post：话题是 AI agents，100字，中文
帮我发个推，讲讲 agent memory governance
Help me tweet about the new product launch
```

### Raw-text mode

```
发X：我心中的 AI Agent 终极形态…… #AIAgents #LLM
/xpost Just shipped v2.0! #shipping #startup
tweet this: Excited to announce our Series A! #funding
```

## Key technical detail

X’s post composer uses a `contenteditable` Draft.js editor. Synthetic keystroke typing corrupts hashtags (renders as `####`). This skill uses `document.execCommand('insertText')` via `javascript_tool` for reliable text insertion, plus a read-back verification before clicking Post.

## Adapted API mapping

| Original (OpenClaw) | Cowork (Claude in Chrome) |
|---|---|
| `browser.open(profile="user")` | `tabs_context_mcp` + `tabs_create_mcp` + `navigate` |
| `browser.act kind="evaluate"` | `javascript_tool` |
| `browser.snapshot` | `screenshot` / `read_page` / `find` |
| Click interactions | `computer` (left_click) or `find` + `ref` |

## Repo contents

- `SKILL.md` — The Cowork skill (this is what Claude reads)
- `README.md` — This file

## Credits

Adapted from [win4r/x-post-browser-skill](https://github.com/win4r/x-post-browser-skill) for the Claude Cowork ecosystem.

## License

MIT
