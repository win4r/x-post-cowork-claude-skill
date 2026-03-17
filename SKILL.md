---
name: x-post-browser
description: "Publish posts to X (Twitter) via Claude in Chrome browser automation (x.com web UI, not API). Use when the user asks to post/tweet on X, publish to x.com, send a tweet, or says '发 X post' / '发推', and wants Claude to open x.com and click Post. Supports two modes: (1) topic mode — Claude drafts a post then waits for user confirmation before publishing, (2) raw-text mode — user provides the final post text, Claude publishes immediately. Also trigger when the user says 'help me post on Twitter', 'publish this to X', 'tweet this', or any variation of wanting to send a post through the X/Twitter web interface."
---

# X Post via Browser (Claude in Chrome)

This skill automates posting to X (Twitter) through Cowork's Claude in Chrome browser tools. It clicks the same UI a human would — no API keys or tokens needed. The user just needs to be logged in to X in their Chrome browser.

## Mode selection

Default to **topic mode** unless the user clearly provided final post text.

**Raw-text mode (publish immediately)** applies when any of these is true:
- The user message starts with a "final text" prefix like `发X：`, `原文：`, `正文：`, `内容：`, `post：`, `tweet：`
- The user uses a command-style trigger like `/xpost ...` where the remainder is the final post
- The user says "按以下内容发布 / 照这个发 / 直接发 / 就发这段" and includes the full text

Otherwise use **topic mode (draft + confirm)**:
- The user gives a topic, angle, requirements (length/tone), or asks "写一条 / 帮我发一条"

## Topic mode: drafting rules

- Language: **Chinese** by default (follow user's language preference if specified)
- Length target: **~100 Chinese characters** (stay within X's limits after hashtags)
- End with **3–6 English hashtags** derived from the topic
  - Mix 1–2 broad searchable tags + 2–4 precise tags; avoid irrelevant trend hijacking
- Present the draft to the user and ask for confirmation — user must reply **"发布"** / **"发"** / **"post it"** to proceed
- If the user requests edits, revise and re-confirm before publishing

## Raw-text mode

- Use the user-provided text **verbatim**
- Publish immediately without a second confirmation
- If the text looks ambiguous (more like a topic than final copy), fall back to topic-mode confirmation

## Browser workflow (Claude in Chrome)

The tools used here are Cowork's `mcp__Claude_in_Chrome__*` tools. Every step below uses the user's real Chrome browser profile, so the user must already be logged in to X.

### Step 0: Get a tab

Before anything else, call `tabs_context_mcp` (with `createIfEmpty: true`) to get the current tab group. Then call `tabs_create_mcp` to create a fresh tab for this workflow. Save the returned `tabId` — every subsequent tool call needs it.

### Step 1: Navigate to X home

Use `navigate` with `url: "https://x.com/home"` and the `tabId` from Step 0.

After navigation, take a `screenshot` to confirm the page loaded. If you see a login screen or consent prompt, tell the user: "It looks like you're not logged in to X in this browser. Please log in manually, then tell me to continue." Wait for the user to confirm before proceeding.

### Step 2: Open the compose dialog

The home inline composer can be unreliable. Prefer the dedicated compose dialog:

1. Use `find` with query `"Post button in sidebar"` or `"compose new post button"` to locate the sidebar "Post" button
2. Click it using `computer` with `action: "left_click"` on the found element's coordinates, or use `ref` from the find result
3. Take a `screenshot` to confirm the compose dialog appeared — you should see a text box and a "Post" button inside the dialog

If the dialog doesn't open, try navigating directly to `https://x.com/compose/post` as a fallback.

### Step 3: Insert the post text

X's composer is `contenteditable` (Draft.js-like). Keystroke-based typing can corrupt hashtags (rendering them as `####`). Use DOM insertion via `javascript_tool` instead.

Call `javascript_tool` with this code (replace the text variable with the actual post content):

```js
(() => {
  const text = `YOUR_POST_TEXT_HERE`;

  const el =
    document.querySelector('div[role="dialog"] [data-testid="tweetTextarea_0"]') ||
    document.querySelector('[role="textbox"][aria-label="Post text"]') ||
    document.querySelector('[data-testid="tweetTextarea_0"]');

  if (!el) return { ok: false, err: 'composer textbox not found' };

  el.focus();
  document.execCommand('selectAll', false, null);
  document.execCommand('delete', false, null);
  document.execCommand('insertText', false, text);

  const actual = (el.innerText || '').replace(/\\r/g, '').trim();
  const expected = text.trim();
  if (actual !== expected) {
    document.execCommand('selectAll', false, null);
    document.execCommand('delete', false, null);
    document.execCommand('insertText', false, text);
  }

  return {
    ok: true,
    actual: (el.innerText || '').replace(/\\r/g, '').trim()
  };
})()
```

The reason for `execCommand('insertText')` rather than setting `innerText` directly is that X's Draft.js editor tracks internal state — direct DOM mutation breaks the editor's model and leaves the Post button disabled. `insertText` simulates a real user typing, which keeps the internal state in sync.

### Step 4: Preflight checks

Before clicking Post, verify two things:

**Text integrity** — call `javascript_tool` to read back the composer content:
```js
(() => {
  const el =
    document.querySelector('div[role="dialog"] [data-testid="tweetTextarea_0"]') ||
    document.querySelector('[data-testid="tweetTextarea_0"]');
  return el ? (el.innerText || '').replace(/\\r/g, '').trim() : null;
})()
```
Confirm it matches the intended text and does not contain `####`.

**Button enabled** — call `javascript_tool`:
```js
(() => {
  const btn =
    document.querySelector('div[role="dialog"] [data-testid="tweetButton"]') ||
    document.querySelector('[data-testid="tweetButton"]');
  if (!btn) return { found: false };
  return { found: true, disabled: btn.disabled || btn.getAttribute('aria-disabled') === 'true' };
})()
```
If the button is disabled, re-run the insertion (Step 3) once. If still disabled after retry, stop and report the issue to the user.

Also take a `screenshot` as a visual sanity check before publishing.

### Step 5: Publish

1. Use `find` with query `"Post button in compose dialog"` to locate the tweet button, or use `javascript_tool` to click it:
```js
(() => {
  const btn =
    document.querySelector('div[role="dialog"] [data-testid="tweetButton"]') ||
    document.querySelector('[data-testid="tweetButton"]');
  if (btn) { btn.click(); return { clicked: true }; }
  return { clicked: false, err: 'Post button not found' };
})()
```

2. Wait 3 seconds using `computer` with `action: "wait"`, `duration: 3`

### Step 6: Verify success

1. Take a `screenshot` to see if the dialog has closed (indicating the post was sent)
2. Navigate to `https://x.com/home` and take another `screenshot`
3. Look for the newly posted content near the top of the timeline (often marked "Now" or "1s")
4. If you can find the post, click into it and capture the permalink URL from the browser
5. Report the permalink to the user as confirmation

## Reliability / fallbacks

- **Elements not found**: take a `screenshot`, then use `read_page` with `filter: "interactive"` to see what's actually on the page. Retry once after refreshing.
- **Composer missing**: navigate back to `https://x.com/home`, wait 2 seconds, then try opening the compose dialog again.
- **Hashtags corrupted as `####`**: clear and re-insert using the `javascript_tool` method from Step 3. Never fall back to keystroke typing for hashtags.
- **Post button stays disabled**: the editor's internal state may be out of sync. Try clicking into the textbox first, pressing Ctrl+A then Delete via `computer` `key` action, then re-inserting via `javascript_tool`.
- **Rate limit or error toast**: take a `screenshot`, read the error text, and report it to the user.

## Important notes

- Posting to X is a **public action**. Topic mode includes a confirmation step to reduce accidental posts. Raw-text mode skips this for speed — only use it when the text is final.
- This skill requires the user's Chrome browser to be open and logged in to X. If the browser isn't connected, suggest the user check their Claude in Chrome extension.
- Always ask for explicit user permission before clicking the final "Post" button in topic mode. This is a publishing action.

## Example triggers

- Topic mode:
  - "发布一条 X post：话题是 AI agents，100字，中文"
  - "帮我发个推，讲讲 memory 的治理"
  - "Help me tweet about the new product launch"

- Raw-text mode:
  - "发X：我心中的 OpenClaw 终极形态…… #OpenClaw #AIAgents"
  - "/xpost Just shipped v2.0! #shipping #startup"
  - "tweet this: Excited to announce our Series A! #funding"
