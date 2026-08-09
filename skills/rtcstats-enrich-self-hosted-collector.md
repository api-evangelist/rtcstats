---
name: Enrich sessions from a self-hosted rtcstats collector
description: Use POST /v1.0/enrich to get scores and a flat, SQL-friendly observation list for a dump your own rtcstats-server already holds — without storing the session on rtcstats.com.
api: openapi/rtcstats-api-openapi.yml
operations: [enrich, quota, observations]
generated: '2026-08-09'
method: generated
source: openapi/rtcstats-api-openapi.yml + https://rtcstats.com/llms.txt
---

# Enrich from a self-hosted collector

The data-residency path. The collection layer (`@rtcstats/rtcstats-js` + `rtcstats-server`, MIT) is self-hosted; `enrich` sends a dump to rtcStats for analysis and gets back only the observations-and-scores projection. **The session is never stored on rtcstats.com.**

Use this when you keep WebRTC session data in your own infrastructure and want the analyzer's verdict in your own database.

## Before you start

- `Authorization: Bearer <application JWT>`. Developer plan or above; a plan without API access returns `403`.
- Same one-credit cost as `analyze`. Check `quota` (`GET /v1.0/quota`) first — `402` means no credits.

## Steps

1. **Submit the dump** — call `enrich` (`POST /v1.0/enrich`). Small dumps can go as a raw body. Anything above the ~4.5MB request-body cap uses the same chunked protocol as `upload` and `analyze`: `multipart/form-data` parts (`chunk`, `fileId`, `chunkIndex`) sharing one client-generated `fileId`, then a JSON assemble request `{"assemble": true, "fileId": "..."}`.

2. **Read the response** — `{ data: { scores, observationsCount, observations, userAgentData }, processorVersion, generatedAt }`.
   - `scores` are the five analyzer scores: `experienceScore`, `audioScore`, `videoScore`, `connectivityScore`, `observationsScore`. Each is a number **or null** — handle null before arithmetic.
   - `observationsCount` (`critical`/`high`/`medium`/`low`/`info`) sums to `observations.length`. Use it as a checksum on your insert.
   - `processorVersion` is the `@rtcstats/rtcstats-processor` semver that produced the analysis (e.g. `1.9.0`); `generatedAt` is epoch ms. **Store both** — they are the only way to explain why the same dump scores differently later.

3. **Write the observations to your store.** The enrich projection is deliberately flat and SQL-shaped: only `type` and `severity` are always present. Every other field — `category`, `tags`, `firstSeenAt`, `url`, `property`, `valueInCount`, `valueInPercent`, `averageInMs`, `maxInMs`, `durationInMs`, `durationInSeconds`, `durationInPercent` — is **omitted when absent, never null**. Design the table for sparse columns and treat a missing key as unknown, not zero.

4. **Resolve type names against the catalog** — call `observations` (`GET /v1.0/observations`) periodically and cache it. It returns every `type` the analyzer can emit with its `title`, the full `severity[]` it can carry and its `tags[]`. Join on `type` rather than storing the display title, because titles are presentation and types are the contract.

## Rules

- `enrich` is unconditional and not plan-gated in its payload — you get the full projection regardless of tier (above the API access gate). MOS follows the account token settings.
- Nothing is persisted server-side, so there is **no `rtcstatsId` to fetch later**. If you need a durable session in rtcStats, use `analyze?save=true` or `upload` instead.
- No idempotency key: a retried assemble costs another credit and produces a second analysis. Deduplicate on your side, keyed on your own dump identifier.
- `415` with `errorCode` `parsing_issue` or `other_issue` means the dump was rejected by the parser — re-export it rather than retrying the same bytes.
