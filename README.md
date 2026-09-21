# AwrAIter Agent Skills

Agent skills for working with [AwrAIter](https://awraiter.ai): Telegram channels you administer — analytics, drafts from the channel's own sources, content plans, scheduled publishing — over the [MCP server](https://awraiter.ai/mcp-server/) or the REST v1 API. No userbot: no phone number, no login code.

Read [SKILL.md on GitHub](skills/awraiter/SKILL.md) or fetch the [raw Markdown](https://raw.githubusercontent.com/MediaZione/awraiter-skills/main/skills/awraiter/SKILL.md).

[Русская версия](README.ru.md)

## Install

### Claude Code plugin

```bash
claude plugin marketplace add MediaZione/awraiter-skills
claude plugin install awraiter@awraiter
```

### Other agents via skills.sh

```bash
npx skills add MediaZione/awraiter-skills --skill awraiter
```

Select your agent when prompted. Installation is project-local by default; add `-g` to install globally.

### Manual

Copy `skills/awraiter/SKILL.md` into your agent's skills directory (for Claude Code: `~/.claude/skills/awraiter/SKILL.md`).

## Connect

The skill teaches the agent the tools, the workflow and the guardrails; the connection itself is one of:

- **MCP with OAuth** (Claude, Claude Code, chatgpt.com): add `https://awraiter.ai/mcp` — the client signs you in with Telegram.
- **A personal access token** (REST v1, or MCP clients without OAuth): create one in the panel under [MCP connection](https://awraiter.ai/mcp-connect) and export it:

```bash
export AWRAITER_TOKEN=pat_...
```

Step-by-step for every client: [docs/connect.md](https://github.com/MediaZione/AwrAiter/blob/main/docs/connect.md).

## Use

Ask your agent, for example:

> Show last week's statistics for my channel and draft two posts from its fresh sources; preview them before scheduling.

> Write a Node script that publishes the newest approved draft of channel 123 at 09:00 Rome time, with idempotent retries.

In Claude Code, invoke the skill explicitly with `/awraiter:awraiter`.

| Skill | Purpose |
|---|---|
| [awraiter](skills/awraiter/SKILL.md) | The 25 tools over MCP and REST v1, authentication, the draft → check → preview → schedule workflow, guardrails, where the live docs are |

## License

[MIT](LICENSE).
