---
name: proxyhuman
description: Hand the current browser session to a human when you hit a step that's gated on a person — signing in, solving a captcha, completing 2FA, reading an OTP from email/SMS, entering payment info, picking a subjective option, or any browser interaction where retrying brittle automation is slower than asking. Returns a viewer URL the human opens (you must surface this through your harness's messaging tool — the skill does not notify on its own) and, once they hand control back, a structured log of what they did so you can resume the task.
compatibility: Requires the @proxyhuman/mcp MCP server installed and registered (`npm i -g @proxyhuman/mcp && claude mcp add proxyhuman -- proxyhuman-mcp`). The MCP itself needs Chrome with CDP enabled (Hermes's chrome-shared.service satisfies this out of the box) and network access to https://api.proxyhuman.ai.
metadata:
  author: aidenlabs
  version: "0.1.0"
  homepage: https://proxyhuman.ai
  repository: https://github.com/aidenlabsdotdev/skills
---

## When to use this skill

You're driving a Chrome browser (via Hermes' browser tools, Playwright, Puppeteer, browser-use, or any other CDP-based automation) and you hit a step that's better for a human:

- **Authentication walls** — sign-in pages, 2FA, password resets
- **Anti-bot challenges** — captchas, Cloudflare Turnstile, "are you human" checks
- **Verification flows** — OTP from email/SMS, magic-link emails
- **Payment forms** — credit cards, billing addresses, anything you can't (or shouldn't) auto-fill
- **Subjective decisions** — "which of these search results matches what the user wanted", "which size/color/option to pick"
- **Anything that's failing repeatedly** — if automated retries aren't working, the human is faster than another debug loop

## How to use it (must be in this order)

This skill is a wrapper around the `proxyhuman` MCP server. The MCP exposes two tools you'll call:

### Step 1: Generate a hand-off URL

Call `open_browser_handoff_link` with the CDP endpoint of the browser you need help with:

```
open_browser_handoff_link({
  cdp_target: "http://localhost:9222",     // pass explicitly — the URL of YOUR Chrome
  prompt: "Please sign in with your Google account",   // optional, human-readable
})
→ { sessionId: "...", viewerUrl: "https://app.proxyhuman.ai/browser/..." }
```

The `viewerUrl` is what the human opens — it mirrors the browser to their phone/desktop and lets them tap, scroll, and type inside the same Chrome session you're driving.

### Step 2: **NOTIFY THE HUMAN** (this skill cannot do this for you)

ProxyHuman mints the URL but does NOT push it to the human. You have to send `viewerUrl` to the user via whatever messaging channel your harness owns:

- **Under Hermes** — use Hermes' user-message tool (SMS, Discord, Slack, whatever's configured for this user)
- **Other harnesses** — use whatever notification skill is registered
- If you literally have no way to message the user, **say so back to your caller** and let them handle delivery; calling `wait_for_human_handback` without delivering the URL just times out

Make the message concrete:

> "I need you to sign in to Google — please open this URL on your phone or another tab: `https://app.proxyhuman.ai/browser/abc123`. Click the return-arrow button when you're done."

### Step 3: Wait for the handback

Block on `wait_for_human_handback` with the sessionId. Pick a reasonable timeout (default 600s = 10min):

```
wait_for_human_handback({ sessionId: "...", timeoutSec: 600 })
→ {
    outcome: "human_done" | "timeout" | "disconnected",
    currentUrl: "...",           // the page the human ended on
    actions: [                    // grouped play-by-play of what they did
      { type: "type", text: "..." },
      { type: "press", key: "Enter" },
      { type: "tap", x: 0.5, y: 0.3 },
      { type: "url_update", url: "..." },
      ...
    ]
  }
```

### Step 4: Continue your task

Use `currentUrl` and `actions` to figure out what happened, then resume your work. Common patterns:

- **Auth flow** — you're now signed in; navigate to whatever you were trying to reach
- **Captcha** — the page advanced; carry on with the next step
- **Subjective choice** — read the final URL or last actions to learn which option the human picked

## Prerequisites (one-time setup)

If `open_browser_handoff_link` is not in your tool list, the ProxyHuman MCP server isn't installed. The user needs to:

```bash
npm i -g @proxyhuman/mcp                          # one-time install
proxyhuman sign-up --email <their-email>                 # one-time, sends OTP
proxyhuman verify <otp-from-email>                       # one-time
claude mcp add proxyhuman -- proxyhuman-mcp              # registers the MCP
```

After that the tools appear and this skill works without further config.

## Notes

- The `cdp_target` argument should be the **Chrome you're currently driving**. ProxyHuman has fallbacks (it reads `~/.hermes/config.yaml` and probes common ports) but passing it explicitly is the right pattern.
- The hand-off URL is unguessable, so it's safe to send over SMS/Discord/Slack. It also expires when the session ends.
- The action log uses normalized [0,1] coordinates for taps/scrolls — independent of viewport size.
- Cost is proportional to how long the human is connected (one WebRTC stream from the browser); typical hand-offs are under a minute.
