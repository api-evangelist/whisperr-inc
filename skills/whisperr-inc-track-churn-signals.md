---
name: whisperr-track-churn-signals
description: >-
  Send product events ("churn signals") to Whisperr reliably — batching,
  per-event idempotency, and the retain-vs-drop retry contract. Use when
  instrumenting a backend or client to feed Whisperr's retention engine.
generated: '2026-08-13'
method: generated
source: >-
  openapi/whisperr-inc-runtime-openapi.json (operationIds verified against the
  served spec), https://docs.whisperr.net/api/events/,
  https://docs.whisperr.net/api/delivery/,
  https://docs.whisperr.net/concepts/events/
api: Whisperr Runtime API
base_url: https://api.whisperr.net
operations:
  - trackEventBatch
  - trackEvent
---

# Track churn signals in Whisperr

## Before you start

- Auth header: `X-API-Key: wrk_...` (or `Authorization: Bearer wrk_...`).
- The key is **publishable**. It can only ingest events for its own app, and it
  is expected to ship in client bundles. Do not treat it as a secret, and do not
  build a secret-rotation flow you do not need.
- Prefer `POST /v1/events/batch` (`trackEventBatch`). Use
  `POST /v1/events/track` (`trackEvent`) only for one-off calls.

## Step 1 — Build the event

```json
{
  "external_user_id": "user_8842",
  "event_type": "payment_failed",
  "occurred_at": "2026-06-14T12:00:00.000Z",
  "properties": { "amount_cents": 4900 },
  "context": { "$message_id": "f7a1c2e0-4b3d-4f6a-9c1e-8d2b5a7e3f10" }
}
```

Rules that are enforced server-side, not merely advisory:

- `external_user_id` — your own stable user id, the same one your database uses
  and the same one you pass to `identifyUser`. Frontend and backend events with
  this id land on one timeline.
- `event_type` — lowercase snake_case, matching
  `^[a-z0-9]+(?:_[a-z0-9]+)*$`. Use `object_verb` past tense. Anything else is
  rejected.
- `occurred_at` — RFC3339 UTC, millisecond precision, `Z` suffix. Must be within
  **+5 minutes / −30 days** of now. Capture it when the event happens, not when
  you send it, so buffered or offline events keep their real time.
- `properties` — free-form. Empty serializes as `{}`, never `[]`.
- `context` — free-form but **must** contain `$message_id`.

## Step 2 — Assign `$message_id` exactly once

Generate the id when the event is **created**, before it enters your queue —
never per send attempt. Every retry of the same event must carry the identical
value. The server deduplicates on it, which is the only reason at-least-once
retries are safe.

```
create event ──▶ assign $message_id ──▶ enqueue
                                          │
                     send ◀── retry ◀─────┤   same $message_id every attempt
                                          │
                             2xx ──▶ dequeue
```

If you regenerate it on retry, every timeout becomes a duplicate event.

## Step 3 — Validate before enqueueing

The API rejects unknown fields, so a misspelled key fails the **whole request**.
Batches are capped at **500 events**, and one malformed event 400s the entire
batch. Validate `event_type` and the field names at the point events enter your
queue, so a single bad event cannot wedge the pipeline.

## Step 4 — Send

```bash
curl -X POST https://api.whisperr.net/v1/events/batch \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $WHISPERR_API_KEY" \
  -d '{"events":[ /* up to 500 event objects */ ]}'
```

## Step 5 — Handle the response by class, not by code

| Response | Class | What to do |
|---|---|---|
| 2xx | ok | Delivered. Dequeue the batch. |
| 401 / 403 | auth | **Stop sending.** Surface the error and **keep the batch queued** — retrying will not help until the key is fixed. |
| 429, 5xx, network error, timeout | retry | Bounded exponential backoff. Honor `Retry-After` on a 429 when present. If retries are exhausted, **keep the batch queued** for a later flush. |
| any other 4xx | drop | Malformed and will never succeed. **Drop and log** — this is a bug in your integration. |

Internalize the asymmetry: auth failures and exhausted retries **retain** the
data; malformed payloads **drop** it. Resending identical bytes after a 400
fails identically and only burns quota.

Suggested defaults, matching the official SDKs: ~6 attempts, 10s per-request
timeout.

## Step 6 — Send events that actually matter

Whisperr is a churn engine, not an analytics warehouse. High-signal events,
mostly backend:

- Payment health — `payment_failed`, `card_expiring`, `invoice_overdue`
- Lifecycle — `trial_started`, `trial_expired`, `subscription_cancelled`, `plan_downgraded`
- Engagement — `report_generated`, `project_created`, `checkout_completed`
- Friction — `export_failed`, `support_ticket_opened`

Only event types configured during onboarding drive interventions. Others are
accepted but inert, so you can instrument broadly now and wire them up later.

## Verify your client

If you are not using an official SDK, test against the provider's own
fixtures — `conformance/wire.json` and `conformance/behavior.json` in
[WhisperrAI/whisperr-spec](https://github.com/WhisperrAI/whisperr-spec). They
are the same fixtures every official SDK runs in CI.
