# aidenlabsdotdev/skills

[Agent Skills](https://agentskills.io) published by
[Aiden Labs](https://aidenlabs.dev). Install into any
agentskills.io-compatible harness (Claude Code, Hermes, Codex, Cursor,
Cline, browser-use, …) via the [`skills`](https://www.npmjs.com/package/skills)
CLI.

## Available skills

| Skill         | What it does                                                                                                                                            |
|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| `proxyhuman`  | Hand the current browser to a human for sign-in / captcha / OTP / payment / subjective decisions, then resume with a structured log of what they did. Wraps the [`@proxyhuman/mcp`](https://www.npmjs.com/package/@proxyhuman/mcp) MCP server. |

## Install

```bash
npx skills add aidenlabsdotdev/skills --skill proxyhuman --agent claude-code
# or --agent hermes-agent, --agent codex, --agent cursor, etc.
```

That drops the right `SKILL.md` into your harness's skills directory
(e.g. `~/.claude/skills/proxyhuman/` for Claude Code,
`~/.hermes/skills/proxyhuman/` for Hermes). See the
[`skills` package docs](https://www.npmjs.com/package/skills) for the
full list of supported `--agent` values and where each one writes.

Most skills here also require an MCP server or other backing service —
see the individual `SKILL.md` for prerequisites.

## Layout

```
proxyhuman/
  SKILL.md     # frontmatter (name, description, compatibility) + body
```

Each top-level directory is one skill — same flat layout as
[`agentmail-to/agentmail-skills`](https://github.com/agentmail-to/agentmail-skills).
The folder name is what you pass to `--skill`.
