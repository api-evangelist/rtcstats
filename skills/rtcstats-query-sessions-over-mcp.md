---
name: Query rtcStats sessions over MCP
description: Wire an agent to the hosted rtcStats MCP server and use get_quota, list_sessions and get_session — including what the MCP surface deliberately cannot do.
api: openapi/rtcstats-api-openapi.yml
operations: [mcpStreamablePost, quota, listSessions, session]
generated: '2026-08-09'
method: generated
source: mcp/rtcstats-mcp.yml + mcp/rtcstats-tool-crosswalk.yml + https://rtcstats.com/integrations/mcp
---

# Query rtcStats over MCP

rtcStats runs a first-party hosted MCP server at `https://api.rtcstats.com/v1.0/mcp` — Streamable HTTP, stateless JSON-RPC 2.0, server `rtcstats` v1.9.0, protocol version `2025-06-18`.

## Connect

- Endpoint: `https://api.rtcstats.com/v1.0/mcp` (POST).
- Auth: `Authorization: Bearer <application JWT>` as an HTTP header on the MCP requests. Token from **Settings > Applications** in the dashboard, shown once.
- Plan: Developer or above.
- `initialize` and `tools/list` answer **without** a token, so you can discover the tool contract before provisioning credentials. `tools/call` requires the Bearer header.

## Tools

1. **`get_quota`** — no input. Returns `accountName`, `accountPlan` (`free` | `developer` | `trial_developer` | `enterprise`), `allowedCredits`, `remainingCredits`. Backed by `quota` (`GET /v1.0/quota`). Call this first if you are about to reason about capacity.

2. **`list_sessions`** — the discovery tool. Fifteen optional filters, all mirroring the REST query parameters on `listSessions`: `name`, `observationTypes`, `observationTags`, `os`, `browser`, `browserVersion`, `userId`, `conferenceId`, `sessionId`, `hasCritical`, `hasHigh`, `hasMedium`, `hasLowScore`, `hasMediumScore`, `hasHighScore`. Different filters AND together; multiple values within one filter OR together. Returns `{total, data[]}` where each row carries `rtcstatsId`, `createdAt`, `sessionStart`, `sessionEnd`, `title`, `rtcstatsUrl`, `embedUrl` (Enterprise only) and the denormalized `abstract`.

3. **`get_session`** — input `{rtcstatsId: <uuid>}`. Returns `{rtcstatsId, rtcstatsUrl, embedUrl, data}` with the full analysis. Backed by `session` (`GET /v1.0/sessions/{rtcstatsId}`).

All three are **read-only and consume no analyze credits**.

## What MCP cannot do

The MCP surface is a strict subset of the REST API. There is **no** tool for:

- `upload`, `analyze` or `enrich` — you cannot submit a dump over MCP. Ingest is REST-only.
- `deleteSession` — deliberately excluded; all MCP tools are read-only.
- `observations` — the observation-type catalog has no tool, even though `list_sessions` filters depend on those exact type names.

**Consequence for the agent:** to build a valid `observationTypes` or `observationTags` filter, read the type names from `https://rtcstats.com/llms.txt` (which documents the tag vocabulary) or call `GET /v1.0/observations` over REST with the same Bearer token. Do not invent type names — an unknown type silently matches nothing rather than erroring.

## Rules

- Only critical/high/medium observations are indexed, so `observationTypes`/`observationTags` filters cannot surface a low- or info-severity finding.
- `abstract` is `null` on rows not yet indexed; branch on that before reading `abstract.scoreBand`.
- `embedUrl` is absent (not null) on non-Enterprise plans.
- `data.aiSummary` is `null` until background generation finishes and on plans without the AI feature — re-call `get_session` rather than treating null as "no problems found".
- MCP transport failures come back as a JSON-RPC envelope with `id: null` at HTTP 400/404. **Tool** failures arrive at HTTP 200 inside the stream as a JSON-RPC response carrying `error: {code, message, data}` — a 200 is not proof of success.
