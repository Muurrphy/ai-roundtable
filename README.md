# AI Roundtable

A [Claude skill](https://docs.claude.com) that lets Claude collaborate with the **web version of ChatGPT** through the Claude in Chrome extension — using your existing ChatGPT subscription, **no API keys, no API bills**.

> 让 Claude 和网页版 ChatGPT 开圆桌会:Claude 通过浏览器扩展驱动你已登录的 ChatGPT,轮流讨论、互相评审、收敛后再动手——全程不用复制粘贴,花的是你已有的订阅额度而不是 API 费用。

## Why

Two AIs genuinely sharpen each other: one catches the other's blind spots, and a disagreement forced into the open usually surfaces the real crux. But doing this by hand means endless copy-pasting between tabs. APIs solve that — at metered prices. This skill takes the third path: Claude drives the ChatGPT web UI in your logged-in browser, acting as participant, facilitator, and scribe.

## What makes it more than a message-ferry

- **Turn-based roundtable** — one AI acts at a time; discuss → cross-review → converge → only then implement.
- **Grounding loop** — Claude can run code, read files, and check docs, so checkable claims from either AI get *verified against reality* before the next round. Two LLMs left alone will happily agree on something wrong; reality in the loop fixes that.
- **Anti-echo-chamber protocol** — the conversation opens inside your ChatGPT Project (so ChatGPT brings its own context, not just Claude's framing), and Claude asks for a blind independent diagnosis *before* revealing its own position.
- **A running ledger** — positions, agreements, open questions, and evidence, so round 4 remembers what round 2 settled.
- **Field-tested browser mechanics** — completion detection for reasoning models, JS text injection (typing long mixed-language text corrupts), and recovery from the permission-modal-kills-the-stream failure, all learned in real sessions and documented in `references/chatgpt-web.md`.

## Install

Claude Desktop (Cowork): Settings → Capabilities → Skills → import the `ai-roundtable` folder (or a packaged `.skill` file).

Claude Code: copy the `ai-roundtable/` folder into your skills directory (e.g. `~/.claude/skills/`).

Requirements: Claude with the **Claude in Chrome** extension connected, and a browser where you're logged into ChatGPT.

## Use

Just say something like:

- "让 ChatGPT 一起看看这个问题" / "Get a second opinion from ChatGPT"
- "你们俩讨论收敛之后再动手" / "Debate this with ChatGPT before we implement"

Claude opens ChatGPT in your browser, runs the rounds, verifies claims, and reports back with its own take. You watch and decide; you never copy-paste.

## Honest notes

- Automating the ChatGPT web UI sits in tension with OpenAI's terms of use. This tool exists for personal, attended use of your own account; use your own judgment.
- Browser rounds are slower than API calls (tens of seconds per round). This is a deliberate trade: zero marginal cost, at the speed of patience. Built for hard problems, not rapid-fire chat.
- Works with other web AIs (Gemini, DeepSeek, Kimi, ...) with minor adaptation — see the references.

## License

MIT

