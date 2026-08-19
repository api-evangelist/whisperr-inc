---
name: whisperr-identify-users-and-consent
description: >-
  Tell Whisperr who a user is and how it may reach them — traits, email/sms/push
  channels, and opt-in state — then read back the user and their computed state.
  Use when wiring login/session flows or managing contact consent.
generated: '2026-08-13'
method: generated
source: >-
  openapi/whisperr-inc-runtime-openapi.json (operationIds verified against the
  served spec), https://docs.whisperr.net/api/identify/,
  https://docs.whisperr.net/api/overview/
api: Whisperr Runtime API
base_url: https://api.whisperr.net
operations:
  - identifyUser
  - getUser
  - getUserState
---

# Identify users and manage contact consent in Whisperr

`POST /v1/identify` (`identifyUser`) is idempotent and safe to call on every
login or session restore. Traits are merged server-side, so you can send a
partial object.

## Step 1 — Build the identify object

```json
{
  "external_user_id": "user_8842",
  "traits": { "plan": "pro" },
  "preferred_channel": "email",
  "channels": [
    { "channel": "email", "address": "person@example.com", "opted_in": true, "verified": false }
  ]
}
```

- `external_user_id` (**required**) — the same stable id you send on events.
  This is the join key; Whisperr mints no id of its own.
- `traits` — free-form attributes, merged server-side. Omit when empty.
- `preferred_channel` — `email` | `sms` | `push`. **Optional, and usually leave
  it unset**: by default Whisperr picks a channel based on engagement. Set it
  only to record an explicit user choice.
- `channels` — how Whisperr may reach the user, and whether it is allowed to.

## Step 2 — Get the channel fields right

| Field | Type | Notes |
|---|---|---|
| `channel` | string, required | `email` \| `sms` \| `push` |
| `address` | string, required | Email address, E.164 phone number, or push token |
| `opted_in` | bool | Defaults to `true`. Set `false` to record an opt-out — Whisperr will not use that channel. |
| `verified` | bool, optional | Omit unless you verified the address yourself. |

**The single most common integration bug:** the field is named `channel`, not
`type`. Because the API rejects unknown fields, sending `type` fails the whole
request with a 400 rather than being ignored.

## Step 3 — Treat `opted_in` as consent, not configuration

`opted_in: false` is how an opt-out is recorded, and it is the flag that stops
Whisperr contacting a user on that channel. Propagate unsubscribes and
preference-centre changes here promptly. Set `verified` only when you actually
performed verification — do not default it to `true`.

Note that this endpoint carries direct personal data: email addresses, phone
numbers and push tokens. Whisperr publishes no compliance certifications, trust
center or security.txt, so confirm your own DPA and data-residency requirements
with the vendor directly before sending production identifiers.

## Step 4 — Send

```bash
curl -X POST https://api.whisperr.net/v1/identify \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $WHISPERR_API_KEY" \
  -d '{"external_user_id":"user_8842","traits":{"plan":"pro"},
       "channels":[{"channel":"email","address":"person@example.com","opted_in":true}]}'
```

If you use an official SDK, the `email` / `phone` / `pushToken` conveniences
expand to opted-in `email` / `sms` / `push` channels and produce the identical
wire request:

```js
whisperr.identify("user_8842", { email: "person@example.com", traits: { plan: "pro" } });
```

## Step 5 — Read the user back

- `GET /v1/users/{external_id}` (`getUser`) — the stored user record.
- `GET /v1/users/{external_id}/state` (`getUserState`) — the computed runtime
  state Whisperr's decisioning uses.

Both take **your** external id in the path, not a Whisperr-minted id.

## Step 6 — Handle errors

Same classification as ingestion: `401`/`403` are auth (stop and retain),
`429`/`5xx` are transient (backoff, honor `Retry-After`, retain on exhaustion),
any other `4xx` is malformed (fix it — do not retry identical bytes). Errors
return `{"error":{"code","message","request_id"}}`; quote `request_id` when
contacting the vendor.

## Ordering

Call `identifyUser` before, or alongside, the events for that user. Browser and
mobile SDKs buffer anonymous events and retroactively attribute them once
`identify()` runs; if you are calling the HTTP API directly, there is no such
buffering — send the identify yourself.
