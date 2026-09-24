# mcp_jeedom_skills

Agent skill for using a [Jeedom](https://www.jeedom.com/) smart home well through the
[mcp_jeedom](https://github.com/limad/jeedom_mcp) MCP server — for Claude Code, Codex, and any
other agent that reads `AGENTS.md`/Claude Skills.

## What it covers

- **Token-efficient queries** — targeted lookups vs full-house dumps, ETag reuse for repeated
  polling, `summary`/`compact` formats, bulk read/execute instead of looping single calls,
  avoiding the ~16K-token full ID map unless truly needed.
- **Scenario authoring** — Jeedom's `trigger_tags` syntax (`#trigger_name#`, `#trigger_id#`,
  `#trigger_value#`, `#trigger#`), the quoting pitfall that breaks conditions silently, and why
  not to fall back to the pre-4.5 `#cmdId#` syntax that general Jeedom knowledge often suggests.
- **The `merged`/`legacy` tool-naming split** — `mcp_jeedom` can expose either grouped
  action-based tools (`state(action=find)`) or one tool per action (`find_command`), depending
  on how the server is configured; the skill gives both names so it stays correct either way.

## Install

As a Claude Code plugin (marketplace add), or by cloning this repo and pointing your agent at
`skills/jeedom-mcp-best-practices/SKILL.md` / `AGENTS.md` directly.

## Scope

This skill is about *using* an mcp_jeedom server well — not about developing Jeedom plugins.
It complements the behavior the `mcp_jeedom` server already enforces at the protocol level
(response integrity, hiding internal IDs from the user, confirmation before sensitive actions)
with what doesn't fit in that always-loaded instructions field.

## License

MIT — see [LICENSE](LICENSE).
