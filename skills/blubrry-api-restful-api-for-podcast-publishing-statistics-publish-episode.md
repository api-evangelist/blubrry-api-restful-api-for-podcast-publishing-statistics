---
name: blubrry-publish-episode
description: Upload a media file to a Blubrry show, publish it, and create the episode that carries it — the full publishing path, with the irreversibility warnings that path requires.
api: Blubrry Podcast Hosting & Statistics API
base_url: https://api.blubrry.com/2
operations:
  - listPrograms
  - uploadMedia
  - listUnpublishedMedia
  - publishMedia
  - addEpisode
  - updateEpisodeImage
  - deleteEpisodeImage
  - deleteMedia
generated: '2026-09-06'
method: generated
source: openapi/blubrry-api-restful-api-for-podcast-publishing-statistics-podcaster-openapi.yaml
---

# Publish an episode on Blubrry

## Before you start

**Read this first — it changes how you should sequence the work.** Blubrry's published contract has **no
delete-episode operation**. Once `addEpisode` succeeds, the episode cannot be removed through the API; only
its image can be deleted, and only the media file behind it can be deleted. If you create the wrong episode,
a human has to go to the Blubrry dashboard. Treat `addEpisode` as the last and most consequential step, and
confirm the media is correct before you take it.

There is also **no idempotency key** on any Blubrry write. If `addEpisode` times out, do not blindly retry —
call `getEpisodes` first and check whether the episode already exists, or you will publish it twice.

## Authentication

Every call needs `Authorization: Bearer <access_token>`. Tokens come from the authorization-code flow at
`https://api.blubrry.com/oauth2/authorize` / `https://api.blubrry.com/oauth2/token` and **expire after one
hour**. Refresh tokens do not expire. There are no scopes: your token can do everything the account can do,
so scope the work yourself.

## Steps

1. **Find the show keyword.** `listPrograms` — `GET /media/index.json`. Every other call in this flow is
   addressed by the show's `keyword`, not by a numeric id. The response also carries `program_id`,
   `podcast_id` and `stats_program_id`; these are three different systems' ids and are not interchangeable.
   The `xTotalHosted` response header tells you how many shows exist.

2. **Upload the media file.** `uploadMedia` — `PUT /media/{keyword}/{mediafile.ext}` with the raw file body.
   For a large file, send it in chunks using the `X-RawVoice-Range` header (start and end bytes). The
   response returns `X-RawVoice-MD5-Checksum` — **verify it against your local checksum before going on**.
   This is one of the genuinely good parts of this API; use it.

3. **Confirm the upload landed.** `listUnpublishedMedia` — `GET /media/{keyword}/index.json`. Your file
   should appear with `name`, `length`, `created` and `last_modified`. If it does not, stop here rather than
   publishing.

4. **Publish the media file.** `publishMedia` — `GET /media/{keyword}/{mediafile.ext}`. Yes, publishing is a
   GET in this API. The response returns `media_url`, which is what the episode will point at.

5. **Create the episode.** `addEpisode` — `POST /episode/{keyword}/add`, body
   **`application/x-www-form-urlencoded`** (Blubrry accepts no JSON bodies). Set `status` to control whether
   this publishes, schedules or saves as draft, and set `filename` to the media file from step 2. The rest of
   the body is an RSS/iTunes podcast item: `title`, `content`, `explicit`, `itunes_title`, `itunes_season`,
   `itunes_episode_number`, `itunes_order`, `release_date`, `episode_type`, `podcast_link`.
   **Prefer `status` = draft or scheduled on a first run** so a human can review before it goes live.
   Success returns `episode_id`, `created_at`, `created_at_timestamp` and `listing_url`.

   Note `itunes_episode` is marked "(Deprecated)" in the contract — use `itunes_title` instead.

6. **Attach artwork (optional).** `updateEpisodeImage` — `PUT /episode/{keyword}/update-image/{episode_id}/`,
   `multipart/form-data` with an `image` part. Returns `episode_image_url` and `episode_image_url_resized`.

## Undoing things

| What you did | How to undo it | Window |
|---|---|---|
| `updateEpisodeImage` | `deleteEpisodeImage` — `DELETE /episode/{keyword}/delete-image/{episode_id}/` | none stated |
| `uploadMedia` | `deleteMedia` — `DELETE /media/{keyword}/{mediafile.ext}` — permanent, no restore | none stated |
| `addEpisode` | **nothing.** No API operation removes an episode. | — |

Blubrry states no time window for any of these. Do not tell a user an episode or file can be recovered.

## Errors

Blubrry does not use RFC 9457. Two envelopes exist: the contract types 400/401 as `{error, code}` and the
live API returns `{"error": 401, "error_description": "..."}`.

- `400 episode_show_invalid` — the keyword is not a show on this account. Re-run `listPrograms`.
- `400 Missing required field` — check the form-encoded body against the AddEpisode schema.
- `403` — the account is not entitled to this show. This per-show check is the only authorization boundary
  in the API, because there are no OAuth scopes.
- `404` — show or episode does not exist.
- No `429` and no `5xx` are declared anywhere, and no rate-limit headers are returned. If you get throttled
  you will not be told why, so pace yourself.
