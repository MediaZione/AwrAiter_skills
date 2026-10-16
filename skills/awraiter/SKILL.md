---
name: awraiter
license: MIT
description: >
  Work with Telegram channels through AwrAIter: read channel analytics (growth, reach,
  ER/ERR, what works for this audience), draft posts from the channel's own scraped
  sources in the channel's tone, check a draft for near-duplicates, score it against the
  channel's history, preview the exact Telegram message, create and fill content plans,
  schedule or publish. Use when a task mentions AwrAIter, a Telegram channel the user
  administers, channel statistics, content plans, or publishing to Telegram without a
  userbot. Two surfaces, same 25 tools: the MCP server for chat clients and agents, the
  REST v1 API for code. Read the live docs and the tool catalog before writing an
  integration.
---

# Work with AwrAIter

AwrAIter runs Telegram channels end to end: a content plan per channel, publishing on
schedule, and the analytics for what was published. It is **not a userbot** — no phone
number, no login code, no `.session` file. Everything goes through the user's AwrAIter
account and reaches only the channels of the team they authorized, with the same
permission checks as the panel. Available on every plan, including the free one
(one channel).

The same 25 tools are exposed twice:

| Surface | For | Address |
| --- | --- | --- |
| **MCP server** (Streamable HTTP) | Claude, ChatGPT, Cursor, Claude Code, any MCP client | `https://awraiter.ai/mcp` |
| **REST v1** | Your own code, scripts, CI, other agents | `https://awraiter.ai/api/v1/<tool_name>` |

Prefer MCP when the user is in a chat client or an agent that speaks MCP. Prefer REST v1
when writing application code — it is the same dispatcher, so nothing differs in
behaviour, only in transport.

## Read the live docs

**The live docs and the tool catalog are the source of truth; this skill gives direction.**

- Product and pricing in one page: [llms.txt](https://awraiter.ai/llms.txt).
- MCP server page: [awraiter.ai/mcp-server](https://awraiter.ai/mcp-server/).
- Connection guide for every client (Claude, Claude Code, ChatGPT, Codex, Gemini CLI,
  Qwen Code, JSON-config clients) and ready-to-paste configs:
  [github.com/MediaZione/AwrAiter](https://github.com/MediaZione/AwrAiter) —
  [docs/connect.md](https://github.com/MediaZione/AwrAiter/blob/main/docs/connect.md),
  [docs/tools.md](https://github.com/MediaZione/AwrAiter/blob/main/docs/tools.md),
  [examples/](https://github.com/MediaZione/AwrAiter/tree/main/examples).
- OpenAPI for REST v1, no token needed:
  [`/api/v1/openapi.json`](https://awraiter.ai/api/v1/openapi.json) (also `.yaml`) —
  every tool's input and output schema and every error code. Read it before writing REST
  code instead of guessing argument names.
- With a token, `GET /api/v1/tools` returns the catalog **as this token sees it**:
  a `read` token does not see write tools at all.

## Authenticate

| Client | How |
| --- | --- |
| MCP client with OAuth (Claude web/desktop, Claude Code, chatgpt.com) | Add `https://awraiter.ai/mcp`; the client walks the user through the Telegram sign-in. No token to handle. |
| MCP client without OAuth (ChatGPT/Codex desktop form, Gemini CLI, Qwen Code, custom) | Header `Authorization: Bearer <token>`. |
| REST v1 | Header `Authorization: Bearer <token>`. |

Personal access tokens are created in the panel under
[**MCP connection**](https://awraiter.ai/mcp-connect), shown **once**, start with `pat_`,
and come in two scopes: `read` (read-only tools; every write is refused server-side) and
`full` (everything the account can do in the panel). A token is bound to one user and to
the team that was active when it was created; the team's channel limit and plan apply.

Ask the user to export the token as `AWRAITER_TOKEN` and read it from the environment.
Never paste a token into source, config committed to git, or a chat message. Keep it
server-side in web apps.

```bash
export AWRAITER_TOKEN=pat_...
curl -s https://awraiter.ai/api/v1/list_channels -H "Authorization: Bearer $AWRAITER_TOKEN"
```

Claude Code, with a token instead of OAuth:

```bash
claude mcp add --transport http awraiter https://awraiter.ai/mcp \
  --header "Authorization: Bearer $AWRAITER_TOKEN"
```

## REST v1 contract

- **Read tools** (`sideEffect: false`): `GET /api/v1/<tool>?arg=…` — query parameters are
  the arguments, coerced by the tool's JSON schema (`?limit=5` → `5`, repeated keys → arrays).
  `POST` with a JSON body works too.
- **Write tools** (`sideEffect: true`): `POST /api/v1/<tool>` with a JSON object body;
  `GET` answers `405`.
- **Retries**: send `Idempotency-Key` (≤ 200 chars) on writes; the same key returns the
  same result instead of a second post or a second schedule.
- **Confirmation**: a `428` with `{ ok: "needs_confirmation", preview, confirmationToken }`
  means repeat the same call with `confirmationToken` in the body after showing the
  preview to the user.
- **Errors** are one shape, `{ ok: false, code, message, … }`, and the status follows the
  code: `400 invalid_input` / `channel_id_required` · `401 unauthorized` (with an
  RFC 9728 `WWW-Authenticate` challenge) · `402 subscription_required` ·
  `403 read_only_token` / `channel_not_accessible` / `channel_over_plan_limit` /
  `not_team_member` · `404 *_not_found` · `409 duplicate_in_flight` ·
  `422` other domain codes · `429 rate_limited` / `ai_quota_exhausted` /
  `agent_budget_exhausted` · `500 tool_error`. Handle `429` with backoff; do not retry
  `4xx` blindly.
- Every response carries `X-Request-Id`; quote it when reporting a problem.
- Versioning is by path: compatible changes ship inside `/api/v1`; a breaking change
  becomes `/api/v2` and `/api/v1` keeps answering for 12 months after.

```bash
# read: channel stats for the last 30 days
curl -s "https://awraiter.ai/api/v1/get_channel_stats?channelId=123&periodDays=30" \
  -H "Authorization: Bearer $AWRAITER_TOKEN"

# write: save a draft, idempotently
curl -s -X POST https://awraiter.ai/api/v1/create_post_from_text \
  -H "Authorization: Bearer $AWRAITER_TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: draft-2026-09-21-001" \
  -d '{"channelId":123,"title":"…","body":"…"}'
```

```js
// Node 18+ / any fetch runtime
const res = await fetch("https://awraiter.ai/api/v1/schedule_post", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.AWRAITER_TOKEN}`,
    "Content-Type": "application/json",
    "Idempotency-Key": `schedule-${postId}`,
  },
  body: JSON.stringify({ channelId, postId, publishAt: "2026-09-22T09:00:00Z" }),
});
const result = await res.json(); // { ok: true, … } or { ok: false, code, message }
```

## The tools

Names are identical over MCP and REST. Scope in brackets.

- **Teams and channels** — `list_teams` [read], `switch_team` [full], `list_channels` [read],
  `get_channel` [read]. Start every session with `list_channels`: every other tool takes a
  `channelId` from it.
- **Analytics** — `get_channel_stats` (`channelId`, `periodDays` 1–90) [read],
  `get_channel_insights` (what works for *this* audience against its own 30-day median)
  [read], `get_recent_posts` [read], `search_channel_content` (semantic, over the
  channel's own posts and sources) [read], `deep_analyze` (`channelIds`, `mode`
  `analyze`|`compare`) [full] — **the one tool that spends the team's AI credits**;
  ask before calling it.
- **Sources and drafting** — `search_sources` [read], `draft_from_sources` (`limit` 1–10
  fresh, unused sources — raw material to write from) [read], `check_uniqueness` (`text`)
  [read], `score_post` (`text` or `postId`) [read], `preview_post` [read].
- **Publishing** — `create_post_from_text` (`channelId`, `body`, optional `title`,
  `imageUrls`/`videoUrls`/`audioUrls` as public direct links — all-or-nothing, `deslop` to
  strip AI-tell typography, `planItemId` to fill a plan slot) [full], `schedule_post`
  (`postId`, optional `publishAt` ISO + `zone`; omit to publish now) [full],
  `cancel_scheduled_post` [full].
- **Content plans** — `get_content_plan` [read], `create_content_plan` [full],
  `find_free_slot` [read].
- **Channel memory** — `get_channel_memory` (brand brief + banlist: what keeps posts
  sounding like the channel, not like a model) [read], `set_brand_brief` [full],
  `add_banlist_entry` / `remove_banlist_entry` [full].
- **Autopilot** — `set_autopilot_policy` [full]: `manual` / `streak` / `auto` approval of
  agent-made posts. Leave it alone unless the user asks.
- **Playbooks** — `get_playbook` (`task`: `onboarding`, `analyze`, `create_plan`, `fill`,
  `improve`, `rewrite`, `team`) [read] returns the step-by-step operator workflow for that task,
  written by the product. **Call it first for any multi-step job** rather than improvising the
  order. `onboarding` is the first session with a channel (history, audience, brief, rubrics and
  rhythm, interlinking posts, plan); offer it when a channel has no brand brief yet.

## Attach media

- Links must download the file itself, not open a page. Dropbox: change `dl=0` to `dl=1`.
  Google Drive: `https://drive.google.com/uc?export=download&id=<file id>`, shared by link.
  A file the user drops into this chat does not reach the server — ask for a link.
- Up to 10 images (JPG/PNG/WebP), 3 videos, or 5 tracks (MP3/M4A/OGG/WAV/FLAC, ≤49 MB).
- Tracks (`audioUrls`) go out as music; several form an album with the post text as the
  caption. They cannot share a post with images or videos — make a separate post.
- `audioUrls` needs `rightsConfirmed: true`. Ask the user whether they hold the rights to
  publish those tracks; set it only on a clear yes. Never assume.
- A YouTube or SoundCloud page fails with `media_download_failed`. Tell the user to use
  **Add media → Download audio by URL** in the AwrAIter editor instead.

## Write a post the way the product expects

1. `get_channel` + `get_channel_memory` — language, brand brief, banlist. Write in the
   channel's language and tone; never use a banlisted word.
2. `draft_from_sources` — write from the channel's own fresh sources, not from memory.
   Facts only from the source; no invented numbers, names, dates.
3. `check_uniqueness` on the draft — a near-duplicate of an existing post or a copy of the
   source is not publishable; rewrite.
4. `score_post` — a data-grounded signal for *this* channel; treat `revise` as a request
   to rewrite, not as a number to report.
5. `create_post_from_text` → `preview_post` — show the user the exact message.
6. `schedule_post` only after the user has seen the preview and said yes; use
   `find_free_slot` or the plan's slot when there is a plan. Publishing is a real,
   outward-facing action on a real channel.

Read tokens cannot do steps 5–6; say so instead of failing repeatedly.

## Guardrails

- Never publish, schedule, cancel, change the brand brief, the banlist or the autopilot
  policy without an explicit go-ahead in this conversation.
- Never call `deep_analyze` without saying it costs credits.
- One channel on the free plan; `channel_over_plan_limit` and `subscription_required`
  are answers for the user, not errors to retry.
- The assistant sees only the channels of the authorized team; a `channel_not_accessible`
  means the token's team cannot reach that channel — check `list_teams` / `switch_team`,
  do not guess ids.
- Keep the token in the environment; do not log it, echo it, or write it to files.
