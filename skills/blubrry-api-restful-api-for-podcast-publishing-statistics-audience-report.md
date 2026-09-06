---
name: blubrry-audience-report
description: Pull a podcast's download and play statistics from Blubrry — show totals, per-episode retention, and the country/app/device breakdowns — and explain what the IAB-certified numbers do and do not cover.
api: Blubrry Podcast Hosting & Statistics API
base_url: https://api.blubrry.com/2
operations:
  - listStatsPrograms
  - Summary
  - keyword
  - episodes
  - show-episodes
  - Show Countries
  - Apps
  - App Types
  - Episode Countries
  - Episode Apps
  - Episode App Types
  - Widget
generated: '2026-09-06'
method: generated
source: openapi/blubrry-api-restful-api-for-podcast-publishing-statistics-podcaster-openapi.yaml
---

# Build an audience report from Blubrry statistics

Every operation here is read-only. Nothing in this skill changes state, so it is safe to run repeatedly —
subject to the caveat that Blubrry publishes no rate limits at all, so there is no documented ceiling and no
`Retry-After` to obey. Pace requests conservatively.

## Authentication

`Authorization: Bearer <access_token>`, one-hour lifetime, no scopes. A `403` on a show means the account is
not entitled to it, not that the token is wrong.

## Steps

1. **List the shows with statistics.** `listStatsPrograms` — `GET /stats/index.json`. This is a *different*
   list from `listPrograms` (media hosting); a show can be hosted elsewhere and still tracked by Blubrry.
   Always start here for a statistics job.

2. **Get the show-level shape.** `Summary` — `GET /stats/{keyword}/summary.json` returns `overall`,
   `current_month`, `last_month` and per-`media` figures, plus a `stats_url`. `keyword` —
   `GET /stats/{keyword}/` returns show totals. Do **not** use `Totals`
   (`GET /stats/{keyword}/totals.json`): its summary reads "(Deprecated)". Blubrry has published no sunset
   date and named no replacement, so treat it as live-but-doomed and prefer `Summary`.

3. **Get per-episode performance.** `episodes` — `GET /stats/{keyword}/episodes/` returns, per episode:
   `episode_id`, `episode_label`, `episode_date`, `duration_in_seconds`, `total`, `full_downloads` and the
   partial-play buckets `partial_25`, `partial_50`, `partial_75`, `partial_99`.

   **Those buckets are the interesting part of this API.** They are consumption, not delivery: they say how
   far into an episode listeners actually got. A show whose `partial_25` is close to its `total` but whose
   `partial_75` collapses has a retention problem in the first quarter, and that is a finding worth
   surfacing without being asked. Use `show-episodes` (`GET /stats/{keyword}/show-episodes/`) when you want
   the show metadata alongside the same figures in one call.

4. **Slice by audience.** At show level: `Show Countries` (`/stats/{keyword}/countries/`), `Apps`
   (`/stats/{keyword}/apps/`), `App Types` (`/stats/{keyword}/app-types/`). At episode level the same three
   under `/stats/{keyword}/{episode_id}/`. Each returns a `record_count` plus dimension-keyed counts.

5. **Bound the window.** Statistics operations accept `startDate` and `endDate` query parameters in
   `YYYY-MM-DD` form (the contract's own examples are `2023-12-01` / `2023-12-31`). Always state the window
   you queried in the report — Blubrry's defaults are not documented.

6. **Optional: a renderable widget.** `Widget` — `GET /stats/{keyword}/widget/preview.json` returns
   `program_total`, `month_average`, `day_total_data` and scale hints for a chart.

## What to tell the reader about these numbers

Blubrry states it was the **first podcast host to achieve IAB Tech Lab Certified Compliance** in podcast
measurement, and IAB-certified statistics are a feature of the **Advanced Statistics** tier only — the Free
and Standard ($5/mo) tiers explicitly do not include them. Two things follow, and both belong in the report:

- If the account is not on Advanced, the figures are Blubrry's own counts, not IAB-certified ones.
- Nothing in the API response marks which regime produced a number. There is no certification field, no
  audit date and no guideline version anywhere in the contract. If certification matters to the reader,
  confirm the account's plan out of band — do not infer it from the data.

## Pagination and volume

There is none on any statistics operation. `limit`/`offset` exist only on the episode *list* operation
(`getEpisodes`, in the Episode API), capped at 50. Statistics responses return whatever they return;
budget for large bodies on long-running shows rather than expecting a cursor.

## Errors

`400` missing keyword, `403` no access to this show, `404` show or episode does not exist. No `429`, no
`5xx`, no rate-limit headers, and no `application/problem+json` — the envelope is `{error, code}` in the
contract and `{"error": <int>, "error_description": <string>}` live.
