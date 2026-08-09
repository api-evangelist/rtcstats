---
name: Triage bad WebRTC sessions in rtcStats
description: Find the sessions that actually went wrong — filter stored rtcStats sessions by observation type, tag, severity, score band, browser/OS or customer identifier, then pull the full analysis for the worst ones.
api: openapi/rtcstats-api-openapi.yml
operations: [observations, listSessions, session, deleteSession]
generated: '2026-08-09'
method: generated
source: openapi/rtcstats-api-openapi.yml + https://rtcstats.com/api-docs.md
---

# Triage bad WebRTC sessions

The query loop over already-analyzed sessions. Everything here is read-only and **costs no credits**.

## Before you start

- `Authorization: Bearer <application JWT>` on every call. Developer plan or above.
- Retention is 1 month on Free and 3 months on Developer/Enterprise — sessions older than that are gone.

## Steps

1. **Load the observation catalog first** — call `observations` (`GET /v1.0/observations`). It returns `{total, data: [{type, title, severity[], tags[]}]}`. The `type` values are the exact value space of the `observationTypes` filter, and the `tags` are the value space of `observationTags`. Never guess a type name; read it from here.

2. **Filter the session list** — call `listSessions` (`GET /v1.0/sessions`). All fifteen filters are query parameters:
   - Content: `name` (title substring, ANY of, max 20), `observationTypes` (exact type names, ANY of, max 20), `observationTags` (`connectivity`, `security`, `audio`, `video`, `datachannel`, `outbound`, `inbound`, `peripheral`, `behavior`, `network`, `configuration`, `cpu`, `bug`).
   - Client: `os`, `browser`, `browserVersion` (major version only, e.g. `142`).
   - Your identifiers: `userId`, `conferenceId`, `sessionId` — exact match, carried in the rtcstats JWT at collection time.
   - Severity: `hasCritical`, `hasHigh`, `hasMedium` (booleans).
   - Score band: `hasLowScore` (< 60), `hasMediumScore` (60–79), `hasHighScore` (>= 80).
   - **Different filters are AND-combined; multiple values inside one filter are OR-combined.** Array filters are repeated params or comma-separated.

3. **Read the list rows.** The response is `{total, data[]}`. Each row has `rtcstatsId`, `createdAt`, `sessionStart`, `sessionEnd` (the last two may be `null`), `title`, `rtcstatsUrl`, `embedUrl` (Enterprise only) and `abstract`.
   - `abstract` is the denormalized index row that drives the filters — browser/browserVersion/os/osVersion, userId/conferenceId/sessionId, experienceScore, `scoreBand` (`low` | `medium` | `high` | `unrated`), connectivity, `observationTypes`/`observationTags`, and `observationsCritical`/`High`/`Medium` counts.
   - `abstract` is `null` for rows not yet indexed. When it is non-null, every key is present.
   - **Only critical/high/medium observations are indexed**, so `observationTypes` and `observationTags` filters will never match a low- or info-severity finding.

4. **There is no pagination.** `listSessions` returns `{total, data[]}` with no `limit`, `offset` or cursor. Narrow the result set with filters rather than paging — a broad query returns everything.

5. **Pull the full analysis** for the sessions you care about — `session` (`GET /v1.0/sessions/{rtcstatsId}`), keyed on the UUID from step 3. This returns the complete payload: per-connection stats (`data.pConnections`), the full observation list with `source` pointers, `data.aggregatedStats`, and `data.aiSummary` once background generation has completed.

6. **Read aggregated metrics by key grammar** rather than by lookup table. Keys are `<metric>_<scope>_<agg>_<unit>`: metric = `jitter|rtt|bitrate|packetLoss|mos`, scope = `in_audio|out_audio|in_video|out_video`, agg = `avg|min|max|p5|p95|stddev`, unit = `ms` (jitter, rtt), `kbps` (bitrate), `percent` (packetLoss), `score` (mos, 0–4.5 where 0 means unusable transport). Example: `rtt_out_audio_avg_ms`. Session-level weighted averages are `jitter_avg_ms` and `rtt_avg_ms`; stream counts are `in_audio_count`, `out_audio_count`, `in_video_count`, `out_video_count`. The former unitless keys (`rtt_out_audio`, `in_audio`) were removed in analysis schema 6.9 — do not fall back to them.

7. **Delete only on explicit instruction** — `deleteSession` (`DELETE /v1.0/sessions/{rtcstatsId}`) is destructive and irreversible. It is deliberately not exposed as an MCP tool. Never call it as part of a cleanup heuristic.

## Rules

- A `404` on `session` or `deleteSession` means the `rtcstatsId` does not belong to this token's account — do not retry with a different id.
- `422` on `session` means a reprocess failed; the session still exists.
- Errors are `{"error", "errorCode"}`, not problem+json.
