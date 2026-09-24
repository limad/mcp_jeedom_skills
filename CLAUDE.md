# CLAUDE.md

Guidance for working in this repository (also read by other agents as `AGENTS.md`, a symlink
to this file — keep one source, do not duplicate content into both).

## What this is

A Claude Code Plugin / Skill distributing best practices for driving a Jeedom smart home
through the [mcp_jeedom](https://github.com/limad/jeedom_mcp) MCP server: token-efficient
query patterns and correct Jeedom scenario/trigger_tags syntax. One skill:
`skills/jeedom-mcp-best-practices/SKILL.md`.

## Source of truth

The skill content is authored in the `mcp_jeedom` plugin itself
(`resources/data/mcp_best_practices.md`), because it must stay accurate against that server's
actual tool surface (`tools_mode=merged` vs `legacy` naming, resource URIs, format/etag
options). The same file is served live by the daemon as the `jeedom://best_practices`
resource, and consumed locally by the `ai_assistant` Jeedom plugin
(`mcp_jeedom::getMcpBestPracticesGuide()`).

`skills/jeedom-mcp-best-practices/SKILL.md` in this repo is that same body with YAML
frontmatter added on top. When the source file changes, copy the body here (the frontmatter
`description` field is the only part that lives only in this repo). There is no build step yet
— this is a manual sync, worth automating once this repo has more than one skill or outside
contributors.

## Skill Authoring Principles

- **Conciseness** — patterns and lookup tables, not tutorials or general MCP/LLM explanations.
- **Symptom-based triggering** — `description` frontmatter should describe observable agent
  situations (about to poll state repeatedly, about to fetch the full house, editing a
  scenario trigger), not just topic keywords.
- **No invented tool names** — this server genuinely has two naming schemes
  (`tools_mode=merged|legacy`); never assume one without checking the tool list actually
  received, and keep both names in any table that references tools.
- **Verify against source** — every concrete claim (token costs, tool/action names, deprecated
  syntax) must be checked against the `mcp_jeedom` daemon source before being written down, not
  inferred from general Jeedom knowledge (training data on Jeedom is often pre-4.5 and
  contradicts current trigger_tags syntax).

## Validation

No CI yet. Before publishing a change: `cat skills/jeedom-mcp-best-practices/SKILL.md` and
confirm the YAML frontmatter parses (has `name` + `description`) and the body still matches
`mcp_jeedom`'s `resources/data/mcp_best_practices.md`.
