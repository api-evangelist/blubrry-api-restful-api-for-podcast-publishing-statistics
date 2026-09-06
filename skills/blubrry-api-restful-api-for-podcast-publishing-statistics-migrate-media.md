---
name: blubrry-migrate-media
description: Move a podcast's existing media files into Blubrry hosting from external URLs, and monitor the migration — the one Blubrry write flow that has a real reversal operation.
api: Blubrry Podcast Hosting & Statistics API
base_url: https://api.blubrry.com/2
operations:
  - listPrograms
  - post_addMigrateMedia
  - migrateMediaStatus
  - post_removeMigrateMedia
  - listUnpublishedMedia
generated: '2026-09-06'
method: generated
source: openapi/blubrry-api-restful-api-for-podcast-publishing-statistics-podcaster-openapi.yaml
---

# Migrate existing media into Blubrry

This is the flow for a show moving to Blubrry from another host: you hand Blubrry the URLs of the existing
media files and it pulls them in. Unlike episode creation, this flow **is** reversible — `migrate_remove`
undoes `migrate_add` — which makes it the safest write surface in the API to automate.

## Authentication

`Authorization: Bearer <access_token>`. One-hour lifetime, account-wide, no scopes.

## Steps

1. **Resolve the show keyword.** `listPrograms` — `GET /media/index.json`.

2. **Queue the URLs.** `post_addMigrateMedia` — `POST /media/{keyword}/migrate_add.json`. The request body is
   form-encoded and the `AddMigrateMedia` schema accepts either a single `url` or a plural `urls` — send
   `urls` for a batch rather than looping, since there is no idempotency key and each call is a fresh
   unprotected write.

3. **Poll the status.** `migrateMediaStatus` — `GET /media/{keyword}/migrate_status.json`. There is no
   webhook, no callback and no event stream anywhere in Blubrry's API, so polling is the only option.
   Blubrry publishes no rate limit, which is *not* permission to poll hard — choose a conservative interval
   (a minute or more) and back off on any non-200.

4. **Verify what arrived.** `listUnpublishedMedia` — `GET /media/{keyword}/index.json` lists the files now in
   the show's storage with `name`, `length`, `created` and `last_modified`. Check `length` against the
   source; a truncated pull is the failure mode worth catching here.

5. **Undo, if you need to.** `post_removeMigrateMedia` — `POST /media/{keyword}/migrate_remove.json`. The
   `RemoveMigrateMedia` schema accepts `url`, `urls` or `ids`, so you can remove by the URL you submitted or
   by the ids the queue assigned. **No time window is documented for this**, so do not promise a caller that
   a migration can be reversed "within N days" — say only that a removal operation exists.

   Note what this does and does not undo: it removes entries from the migration queue. A file that has
   already landed in storage is deleted with `deleteMedia` (`DELETE /media/{keyword}/{mediafile.ext}`), which
   is permanent and has no restore path.

## After migration

Migrated files are unpublished media. To attach one to an episode, continue with the
`blubrry-publish-episode` skill from step 4 (`publishMedia`, then `addEpisode`) — and re-read its warning
that `addEpisode` cannot be undone through the API.

## Errors

`400 Missing keyword`, `400 Missing required field` (check the form-encoded body against `AddMigrateMedia` /
`RemoveMigrateMedia`), `403` no access to this show, `404` show does not exist. No `429`, no `5xx` declared.
