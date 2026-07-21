# Driving ChatGPT's web app through Claude in Chrome

This reference covers the concrete browser mechanics for the roundtable loop.
Web UIs change often, so prefer the *heuristics* here over any exact selector —
when a selector fails, fall back to reading the accessibility tree
(`read_page`) and a screenshot (`computer` with `action: "screenshot"`) to
relocate the element visually.

## Getting a tab

```
tabs_context_mcp { createIfEmpty: true }        # returns tab ids
navigate { url: "https://chatgpt.com" }         # or a specific /c/<id> thread
read_page { tabId, filter: "interactive" }      # see composer + controls
```

Create a fresh tab for a new roundtable rather than hijacking an unrelated tab
the user has open. If the user wants to continue an existing ChatGPT
conversation, navigate to that conversation's URL.

## Confirming the user is logged in

After navigating, read the page. Signs you're **logged in and ready**: a message
composer at the bottom (placeholder like "Ask anything" / "Message ChatGPT"),
and the model name in the top-left. Signs you're **not**: "Log in" / "Sign up"
buttons, or a marketing splash. If not logged in, stop and ask the user to log
in — never type credentials yourself.

## The composer

ChatGPT's input is a `contenteditable` area, historically with
`id="prompt-textarea"` (a ProseMirror editor), not a plain `<textarea>`. To
enter text reliably:

- **Short text:** click the composer to focus it, then `computer { action: "type", text: ... }`.
- **Long text (code, specs, documents):** typing char-by-char is slow and can
  trip auto-formatting. Prefer pasting: write the text to the clipboard, focus
  the composer, then paste with `computer { action: "key", text: "cmd+v" }`
  (macOS) or `ctrl+v`. If a clipboard-write tool isn't available in the browser
  context, `javascript_tool` can set the editor's content directly, or type in
  reasonable chunks.

Avoid pressing Enter mid-message if your text contains newlines you want to
keep — in ChatGPT, Enter sends and Shift+Enter inserts a newline. When pasting
multi-line text this is handled for you; when typing manually, compose the whole
message first, then submit.

## Sending

Two ways: press Enter while the composer is focused, or click the send button
(an up-arrow / paper-plane icon at the composer's right edge, often
`data-testid="send-button"` or `aria-label="Send prompt"`). Clicking the button
is less ambiguous than Enter when the message spans multiple lines. Confirm from
a screenshot that the message posted (your text appears as a user bubble and the
composer clears).

## Detecting when generation is complete — the critical step

ChatGPT **streams** its reply. If you read too early you get a partial answer.
Reliable completion signals, in order of preference:

1. **The stop control disappears.** While generating, the send button is
   replaced by a **"Stop"/"Stop generating"** square button (often
   `aria-label="Stop generating"` or `data-testid="stop-button"`). Poll
   `read_page { filter: "interactive" }` every ~2 seconds; when the stop button
   is gone and the send button (or voice/mic button) is back, generation is
   done.
2. **A "Regenerate"/copy/thumbs affordance appears** under the last assistant
   message — these only render after the turn completes.
3. **Text stabilizes.** As a fallback, `get_page_text` twice ~2s apart; when the
   tail of the last assistant message stops changing, it's finished.

Use a bounded poll (e.g. up to ~60–90s for long answers). If ChatGPT is clearly
still going after that, tell the user rather than hanging. Reasoning models
("thinking") can pause a while before visible text appears — watch for the stop
button, which is present the whole time it's working.

## Extracting only the latest reply

You want the newest assistant turn, not the entire transcript. Options:

- `get_page_text` returns the page's readable text; take the content after your
  last user message. Assistant turns are the blocks not attributed to "You".
- `read_page` with a `ref_id` scoped to the last assistant message container
  (assistant turns often carry `data-message-author-role="assistant"`). Reading
  the last such node gives just that reply.
- If layout makes boundaries unclear, a screenshot plus `get_page_text` together
  usually disambiguate where ChatGPT's latest answer starts and ends.

Strip UI chrome ("Copy", "Regenerate", model labels) from what you relay.

## Field-tested lessons (2026-07-20, real session)

These were all hit in one real session; treat them as defaults, not edge cases.

**Long text: inject, never type.** Typing a 600+ character message with the
`type` action froze the renderer and silently dropped Latin characters and
digits from mixed Chinese/English text. Use `javascript_tool` instead:

```js
const ed = document.querySelector('#prompt-textarea')
        || document.querySelector('[contenteditable="true"]');
ed.focus();
document.execCommand('selectAll', false, null);
document.execCommand('delete', false, null);
document.execCommand('insertText', false, MSG);
ed.textContent.length  // verify count matches what you sent
```

Then send by clicking the button via JS (`[data-testid="send-button"]` or
`button[aria-label*="发送"]`), not by pressing Enter.

**Permission modals kill the generation stream.** An apps/connector permission
popup appeared mid-generation and the reply died after ~56s of thinking with a
single character of output — twice, identically. If a reply comes back
absurdly short after long thinking, look for a modal, dismiss it (e.g. the
"知道了" button), and ask ChatGPT to re-answer. Symptom signature:
`generating:false, lastLen:1`.

**Poll completion cheaply via JS**, not screenshots:

```js
JSON.stringify({
  generating: !!document.querySelector(
    '[data-testid="stop-button"], button[aria-label*="停止"]'),
  turns: document.querySelectorAll(
    '[data-message-author-role="assistant"]').length,
  lastLen: (document.querySelectorAll(
    '[data-message-author-role="assistant"]')?.length
    ? document.querySelectorAll(
        '[data-message-author-role="assistant"]')[
          document.querySelectorAll(
            '[data-message-author-role="assistant"]').length-1
        ].innerText.length : 0)
})
```

Reasoning modes can think 60s+ before any visible text; the `wait` action caps
at 10s per call, so chain several waits between polls.

**Opening a chat inside a ChatGPT Project:** find the project's link in the
sidebar (`read_page` shows it as `/g/g-p-<id>/...`), navigate to
`https://chatgpt.com/g/g-p-<id>/project`, and use the composer there. SPA
`click()` on sidebar divs often silently fails — navigate by URL instead.

**Extract the reply** with
`document.querySelectorAll('[data-message-author-role="assistant"]')` and take
the last node's `innerText`, sliced in chunks if long. `get_page_text` on a
long thread wastes tokens re-reading the whole transcript.

## Common snags

- **Rate / usage limits:** the web app may show "You've reached your limit" or
  downgrade the model. Relay this to the user; it's their quota. Don't try to
  route around it via API.
- **A modal or "Continue"/cookie prompt** can steal focus. Read the page, close
  or accept per the user's privacy preference (decline non-essential), then
  resume. Don't accept terms/consent on the user's behalf without asking.
- **Wrong model selected:** if the user wanted a specific ChatGPT model, check
  the model switcher (top-left) before the first send; switch if needed.
- **Message didn't send:** re-focus the composer and click the send button
  explicitly instead of relying on Enter.
- **Selectors changed:** don't get stuck on a stale `data-testid`. Take a
  screenshot, locate the control visually, and click by coordinates as a last
  resort.

## Minimal round, in tool terms

```
# compose
computer { action: "left_click", coordinate: <composer> }
computer { action: "type", text: "<your message to ChatGPT>" }
computer { action: "left_click", coordinate: <send button> }   # or key: Enter

# wait for completion
loop: read_page { filter:"interactive" } every ~2s until stop-button gone

# read
get_page_text  ->  take the last assistant block

# relay to user, add your take, decide whether to continue
```
