---
name: videoverse-subscribe-to-content-events
description: Subscribe to Magnifi content change events over webhooks and verify their HMAC signatures.
generated: '2026-09-04'
method: generated
source: asyncapi/videoverse-magnifi-webhooks.yml + openapi/videoverse-magnifi-partner-openapi.yml
api: Magnifi Partner Integration API
operations:
  - fetchTopics
  - createSubscription
  - getSubscriptions
  - getSubscriptionBySubscriptionId
  - updateSubscription
  - deleteSubscription
---

# Receive Magnifi content events

Magnifi has two independent webhook mechanisms. Pick the one that matches what you need.

## Which mechanism

**Metadata Delivery** pushes the FULL metadata for a piece of content while a stream is processing.
You enable it by setting `publishingWebhookUrl` when you call `createStream` — there is no
subscription API for it. Payloads look like
`{"data": {"metadata": {...}, "version": 2}, "event": "CLIP_CREATED"}`. You may get one
`CLIP_CREATED` followed by several `CLIP_UPDATED` for the same content, because processing is
multi-stage. Delivery is retried up to **3** times.

**Notifier** tells you WHICH FIELDS changed, not the content. It is subscription-managed and covers
the whole entity. Delivery is retried up to **5** times and repeated failures can disable the
subscription.

## Subscribing to the Notifier

1. `fetchTopics` (`GET /v1/webhook/notifier/topics`) returns the subscribable resource types with
   their available operations and fields:
   - `stream` — `title`, `status`
   - `matchVideo` — `matchVideoTitle`, `status`, `progress`, `videoUrl`
   - `clip` — `clipTitle`, `startTime`, `endTime`, `duration`, `players`, `outcome`, `rating`,
     `transcript`, `videoUrl`, `videoThumbnailUrl`, `aspectRatiosAvailableIn`, plus category-scoped
     custom fields (`batsman`, `bowler` for cricket; `corner` for football)
   - `hlClip` — `hlClipTitle`, `duration`, `players`, `outcome`, `rating`, `videoUrl`, `clips`,
     plus the same custom fields

   Do not subscribe to a field that is not in this list for that resource type — read it, do not
   assume it.

2. `createSubscription` (`POST /v1/webhook/notifier/subscription`) with `resourceType`,
   `operations` (`CREATE` / `UPDATE`), `standardFields`, `customFields`, `webhookUrl`, `secretKey`
   and `description`. Returns `sub_ntf_…` with `isActive` and `failureCount`.

3. `getSubscriptions`, `getSubscriptionBySubscriptionId`, `updateSubscription` and
   `deleteSubscription` manage it afterwards. `PA012` is a missing subscription; `PA013` is an
   inactive one — check `failureCount` before recreating it, because a fresh subscription pointed at
   the same broken endpoint will be disabled again.

## Verifying an event

Both mechanisms sign with **HMAC-SHA256 over the raw request body**, using the secret you supplied,
and send the result in the `x-magnifi-signature` header. Recompute it over the raw bytes — not over
a re-serialised object — and compare in constant time.

Metadata Delivery also supports an optional static header instead: you give Magnifi a header name
and an expected value, and it sends that header on every request. This is weaker than the HMAC and
the documentation calls the HMAC the recommended option.

## The event shape

    {
      "eventId": "evt_ntf_<uuid>",
      "subscriptionId": "sub_ntf_<uuid>",
      "resourceId": "…",
      "resourceType": "clip",
      "operation": "UPDATE",
      "changedStandardFields": ["clipTitle"],
      "version": 2,
      "timestamp": "2025-06-13T09:08:52.017Z"
    }

Deduplicate on `eventId`. Retries mean you WILL see the same event more than once, and the API
offers no idempotency mechanism to lean on.
