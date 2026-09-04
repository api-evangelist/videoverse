---
name: videoverse-ingest-live-stream
description: Ingest a live or recorded feed into Magnifi and collect the AI-generated clips, highlights and highlight clips it produces.
generated: '2026-09-04'
method: generated
source: openapi/videoverse-magnifi-partner-openapi.yml (derived from https://docs.prod.videoverse.dev/)
api: Magnifi Partner Integration API
operations:
  - getPartnerDetails
  - getPartnerCategories
  - getPartnerTemplates
  - getPartnerStreamCustomFields
  - createStream
  - getStreamByStreamId
  - getClipsByStreamId
  - getHighlightsByStreamId
  - getHlClipsByStreamId
  - fetchPresignedUrl
---

# Ingest a stream into Magnifi and collect its content

Every request carries `x-access-key` and `x-access-secret`. The base URL is the
`{PARTNER_BASE_URL}` your Magnifi representative issued with those credentials — it is not
published, so do not guess it.

## 1. Confirm what this key can do

`getPartnerDetails` (`GET /v1/partner/details`) returns the user and the entity (organization or
workspace) the key belongs to. Everything you create below lands inside that entity.

## 2. Read the inputs the stream needs

- `getPartnerCategories` (`GET /v1/partner/categories?page=1&pageSize=10`) — the sports/content
  categories this entity is entitled to. Paginated: `page`, `pageSize`; the envelope returns
  `currentPage`, `pageSize`, `totalItems`, `totalPages`.
- `getPartnerTemplates` (`GET /v1/partner/templates`) — templates grouped as `direct`, `medialive`
  and `reserved_channel`. A template is REQUIRED for ingestion.
- `getPartnerStreamCustomFields` (`GET /v1/partner/stream-fields?category=cricket`) — the custom
  fields you may set in `fields`.

## 3. Create the stream — this step is not reversible

`createStream` (`POST /v1/stream`) with `title`, `url`, `category`, `fireAt`, `templateName`, and
optionally `fields` and `publishingWebhookUrl`. It returns `202` with the `stream` object and its
`streamId`.

Before you call it, know what you are committing to:

- No delete or cancel operation for a stream is published.
- `S010` forbids changing the stream URL after creation, `S011` the `fireAt`, `S013` the template
  and `S014` the category. `updateStreamByStreamId` cannot undo a wrong one.
- `fireAt` must be `"now"` or a future timestamp (`S003`).
- One stream per `matchScheduleId` (`S009`).
- The API has NO idempotency key. If a `createStream` call times out, poll `getAllStreams` with your
  `ext_stream_id` in `filters` before retrying, or you will create a second stream you cannot delete.

## 4. Watch it process

`getStreamByStreamId` (`GET /v1/stream/{streamId}`) returns `status`. Allow up to five minutes for
channel provisioning — `S004` records that a MediaLive resource can take that long, and a
`S004 StreamResourceClearanceFailed` means provisioning did not clear in time.

## 5. Collect the content

- `getClipsByStreamId` (`GET /v1/stream/{streamId}/clips?page=1&pageSize=3`)
- `getHighlightsByStreamId` (`GET /v1/stream/{streamId}/highlights`)
- `getHlClipsByStreamId` (`GET /v1/stream/{streamId}/hlclips`)
- `getMatchVideosByStreamId` (`GET /v1/stream/{streamId}/match-videos`)

`C002` means no clips exist for that stream yet — it is a "not ready" answer as often as a real
error. Do not treat it as fatal on the first poll.

## 6. Fetch a playable URL

`fetchPresignedUrl` (`POST /v1/content/presigned-url`) with `contentId` and `contentType` returns
`videoUrl` and `thumbnailUrl`. `V001`, `V002` and `V003` cover a missing video URL, a missing
thumbnail and a pre-signed URL fetch failure.

Do not use `generatePreSignedUrl` (`POST /v1/partner/getPreSignedUrl`) — the provider marks it
"[TO BE DEPRECATED]" in its own collection.

## Errors

Failures return `{"statusCode": …, "error": {"message": …, "code": …, "metadata": {}}}`. Branch on
`error.code`, not on the message text. The full 57-code catalogue is in
`errors/videoverse-problem-types.yml`.
