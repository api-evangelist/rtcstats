---
name: Analyze a WebRTC dump with rtcStats
description: Submit a webrtc-internals or rtcstats dump to rtcStats and read back the Experience Score, Observations and AI summary — including the chunked upload path for dumps over the request-body cap.
api: openapi/rtcstats-api-openapi.yml
operations: [analyze, upload, session, quota]
generated: '2026-08-09'
method: generated
source: openapi/rtcstats-api-openapi.yml + https://rtcstats.com/api-docs.md
---

# Analyze a WebRTC dump

Turns one exported dump into scores, Observations and a plain-English summary.

## Before you start

- Base URL is `https://api.rtcstats.com`. There is no sandbox or test host — every call runs against production and costs a real credit.
- Auth is a single header: `Authorization: Bearer <application JWT>`. Create the token in the rtcStats dashboard under **Settings > Applications**; it is shown once and cannot be read back.
- API access requires the **Developer** plan or above. Without it you get `403`.
- One successful analyze costs **one credit**. Check the balance first with `quota` (`GET /v1.0/quota`) — it returns `allowedCredits` and `remainingCredits` and costs nothing.

## Steps

1. **Check quota** — call `quota`. If `remainingCredits` is 0, stop: the analyze call will return `402` with `{"error", "errorCode"}`.

2. **Decide the submission path by file size.** The request-body cap is about 4.5MB.
   - Under the cap: call `analyze` (`POST /v1.0/analyze`) with the raw JSON dump as the body.
   - Over the cap: use the chunked protocol on the same operation — send `multipart/form-data` parts with fields `chunk`, `fileId` and `chunkIndex`. Reuse one client-generated `fileId` across every chunk. Each chunk returns `{"success": true}` and consumes **no** credit.

3. **Assemble (chunked path only)** — send a small JSON body (max 1KB) to the same operation: `{"assemble": true, "fileId": "<same id>", "fileName": "<optional>"}`. Assemble detection is shape-based, so this does not collide with a raw dump body. This is the step that runs the pipeline and spends the credit.

4. **Choose whether to persist.** `analyze` does **not** store the session by default. Append `?save=true` to persist it; the response then includes `rtcstatsId` and `rtcstatsUrl`. If you passed `fileName` on a saved assemble request, it becomes the session title and is mirrored back as `data.fileName` / `data.title`.
   - If you want the dump stored as a file first and analyzed as a stored session, use `upload` (`POST /v1.0/upload`) instead — same chunked flow, same one-credit cost.

5. **Read the result.** The response is `{ data, rtcstatsId, rtcstatsUrl, processorVersion, embedUrl }`.
   - `data.experienceScore` is 0–100. `data.audioScore` / `videoScore` may be `null`; `connectivityScore` is always present.
   - `data.observationsCount` is the severity histogram (`critical`, `high`, `medium`, `low`, `info`) and sums to `data.observations.length`.
   - Each `data.observations[]` entry carries `type`, `severity`, `category`, `tags[]` and a `source` pointer (`pid`, `sid`, `cpid`, `ssrcId`, `labelId`). **Do not read `label`** — it is deprecated in favour of `source.ssrcId` / `source.sid` / `source.labelId`.
   - `data.aiSummary` is **always `null` on `analyze`**. To get the AI summary you must save the session and re-read it later with `session` (`GET /v1.0/sessions/{rtcstatsId}`) — generation runs in the background, and it stays `null` on plans without the AI feature.
   - `embedUrl` is omitted entirely on non-Enterprise plans.

6. **Fetch the finished analysis** — if you saved it, poll `session` with the `rtcstatsId` until `data.aiSummary` is non-null.

## Rules

- **No idempotency key exists.** Retrying an assemble or a raw-body `analyze` spends another credit and creates another session. Guard retries yourself; never blind-retry a 5xx on a credit-consuming call.
- **Errors are not RFC 9457.** Every failure is `{"error": "<message>", "errorCode": "<code>"}` — switch on HTTP status, not on a type URI. `415` carries `errorCode` `parsing_issue` (corrupt/unsupported dump) or `other_issue` (invalid file or too-old format). `413` means the dump exceeds the plan's file-size limit.
- **Numbers are pre-rounded.** Timestamps are whole milliseconds; every other number has at most 2 decimals except `audioLevel`. Do not treat them as full precision.
- Only the documented fields are stable; other fields may appear and change without notice.
