---
name: proxyhuman
description: Hand the current browser session to a human when you hit a step that's gated on a person — signing in, solving a captcha, completing 2FA, reading an OTP from email/SMS, entering payment info, picking a subjective option, or any browser interaction where retrying brittle automation is slower than asking. Returns a viewer URL the human opens (you must surface this through your harness's messaging tool — the skill does not notify on its own) and, once they hand control back, a structured log of what they did so you can resume the task.
compatibility: Requires the @proxyhuman/mcp MCP server installed (`npm i -g @proxyhuman/mcp`) and registered with the host harness (Claude Code, Hermes, Cursor, Codex, etc. — see the Prerequisites section below for the harness-agnostic install flow). The MCP needs Chrome with CDP enabled and network access to https://api.proxyhuman.ai.
metadata:
  author: aidenlabs
  version: "0.2.1"
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

## Critical: always pass `cdp_target` explicitly

Before opening a hand-off, you must figure out which CDP endpoint corresponds to the Chrome you're driving and pass it as `cdp_target`. **If you let the MCP guess, you risk attaching to the wrong tab or wrong browser entirely — the human will see a different Chrome than the one you're stuck on, and the hand-off is wasted.**

How to find the right CDP target — pick whichever applies to your situation:

- **Hermes `browser_dispatch` / `browser_*` tools** — the session's CDP URL is part of the daemon record; lift it from there.
- **Playwright/Puppeteer** — the endpoint you used to `connectOverCDP(...)` is the same URL to pass here (e.g. `http://localhost:9222`).
- **browser-use Agent** — read `agent.browser.cdp_url` after the browser is started.
- **A Chrome you launched with `--remote-debugging-port=N`** — pass `http://localhost:N`.
- **A remote Chrome (different host, container, etc.)** — pass that host's CDP URL; ProxyHuman will publish from wherever the MCP runs.

If you genuinely can't determine it, the MCP falls back to probing common ports + reading `~/.hermes/config.yaml`. Treat that as a last resort — log a warning to the user so they know which Chrome got picked up.

## How to use it (must be in this order)

This skill is a wrapper around the `proxyhuman` MCP server. The MCP exposes four tools:

| Tool                          | Purpose                                                                 |
|------------------------------|-------------------------------------------------------------------------|
| `open_browser_handoff_link`  | Mint a fresh hand-off; returns `{ sessionId, viewerUrl }`.              |
| `wait_for_human_handback`    | Block until human clicks "return control" (or timeout). Use this once. |
| `get_handoff_status`         | Non-blocking poll: current state, viewer count, partial action log.    |
| `cancel_handoff_link`        | Abort a pending hand-off you no longer need. Reaches the human if they're connected. |

### Step 1: Generate a hand-off URL

```
open_browser_handoff_link({
  cdp_target: "http://localhost:9222",   // REQUIRED in spirit — see section above
  prompt: "Please sign in with your Google account",   // shown on the dashboard + can frame the ask to the human
})
→ { sessionId: "...", viewerUrl: "https://hmnpr.xyz/..." }
```

The `viewerUrl` is a short link that redirects to `app.proxyhuman.ai/browser/<sessionId>` — it mirrors the browser to the human's phone/desktop and lets them tap, scroll, and type inside the same Chrome session you're driving.

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
    state: "complete" | "failed" | "cancelled",         // terminal state
    outcome: { type: "human_done" }                     // or one of:
            | { type: "timeout" }                       //   no viewer ever connected
            | { type: "disconnected" }                  //   skill WS dropped
            | { type: "cancelled", reason?: "..." }     //   you or admin aborted
            | { type: "cdp_lost" }                      //   the Chrome tab died
            | { type: "encoder_crash" }                 //   ffmpeg died mid-stream
            | { type: "relay_error", detail?: "..." },
    currentUrl: "...",           // the page the human ended on
    actions: [                    // compacted play-by-play of what they did
      { type: "type", text: "...", redacted?: true },
      { type: "paste", text: "..." },
      { type: "press", key: "Enter", modifiers?: ["shift"] },
      { type: "tap", x: 0.5, y: 0.3 },
      { type: "scroll", x: 0.5, y: 0.5, deltaX: 0, deltaY: -120 },
      { type: "navigate", url: "..." },
      { type: "back" } | { type: "forward" } | { type: "reload" },
      { type: "url_update", url: "..." },
      { type: "human_done" },
    ]
  }
```

`redacted: true` appears on `type`/`paste` actions when the publisher detected the human was filling a password / OTP / payment field — the text is replaced with `*` chars before persisting. The behavior is automatic; you don't need to handle it specially.

Session lifecycle (so you can interpret intermediate states from `get_handoff_status`):

```
awaiting_viewer  (URL minted, human hasn't opened it yet — 30min idle auto-cancels)
       │
       ▼
   streaming  ↔  paused  (last viewer left; encoder stops, session alive)
       │
       ▼
  complete / failed / cancelled   (terminal)
```

### Step 4: Continue your task

Use `currentUrl` and `actions` to figure out what happened, then resume your work. Common patterns:

- **Auth flow** — you're now signed in; navigate to whatever you were trying to reach
- **Captcha** — the page advanced; carry on with the next step
- **Subjective choice** — read the final URL or last actions to learn which option the human picked
- **Forensics** — point the user at the dashboard if they want to inspect what happened: `https://app.proxyhuman.ai/sessions/<sessionId>` (or `app.proxyhuman.ai/sessions` for the full list).

### Aborting before handback

If you change your mind mid-flight (you found another path, the user said "never mind", etc.) call `cancel_handoff_link({ sessionId, reason: "..." })`. The viewer's screen reflects the cancel; `wait_for_human_handback` returns with `outcome.type === 'cancelled'`.

## Prerequisites (one-time setup) — run these on the user's behalf if needed

If `open_browser_handoff_link` is **not** in your tool list when you read this
skill, the ProxyHuman MCP server isn't installed yet. Run the install on the
user's behalf — you have shell access, they don't need to leave the chat:

1. **Install the MCP server** (one shell command, ~10s with ffmpeg
   postinstall on top):
   ```bash
   npm i -g @proxyhuman/mcp
   ```

2. **Sign the user up.** Ask them for their email if you don't already have
   it. This sends a one-time code to that address:
   ```bash
   proxyhuman sign-up --email <their-email>
   ```
   Then ask the user to read you the code from their inbox and verify:
   ```bash
   proxyhuman verify <code-they-give-you>
   ```
   The API key is saved to `~/.proxyhuman/config.json` (chmod 600). The MCP
   reads it automatically.

3. **Register the MCP with your harness.** This step is harness-specific —
   *you know your own harness*, so do whatever is correct for it. Examples:

   | Harness        | How to register `proxyhuman-mcp`                                                                                          |
   |----------------|---------------------------------------------------------------------------------------------------------------------------|
   | Claude Code    | `claude mcp add proxyhuman -- proxyhuman-mcp`                                                                             |
   | Hermes         | Add a `proxyhuman:` entry under `mcp_servers:` in `~/.hermes/config.yaml` with `command: proxyhuman-mcp`. Then `hermes reload`. |
   | Cursor / Codex | Add to the harness's MCP config (varies — consult the harness's docs)                                                     |

   After registering, the `open_browser_handoff_link` and
   `wait_for_human_handback` tools become available on your next turn and
   the rest of this skill applies.

If you (the agent) don't have shell access on the user's machine, paste the
three blocks above into the conversation and ask them to run them.

## Notes

- The `cdp_target` argument should be the **Chrome you're currently driving**. ProxyHuman has fallbacks (it reads `~/.hermes/config.yaml` and probes common ports) but passing it explicitly is the right pattern.
- The hand-off URL is unguessable, so it's safe to send over SMS/Discord/Slack. It also expires when the session ends.
- The action log uses normalized [0,1] coordinates for taps/scrolls — independent of viewport size.
- Cost is proportional to how long the human is connected (one WebRTC stream from the browser); typical hand-offs are under a minute.
