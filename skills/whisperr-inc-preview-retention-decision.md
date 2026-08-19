---
name: whisperr-preview-retention-decision
description: >-
  Dry-run Whisperr's decisioning engine — see which retention intervention would
  fire for a user WITHOUT dispatching anything to them. Use when testing,
  evaluating, or letting an agent inspect Whisperr behavior safely.
generated: '2026-08-13'
method: generated
source: >-
  openapi/whisperr-inc-runtime-openapi.json (operationIds and the
  PreviewDecisionRequest/PreviewDecisionResponse schemas verified against the
  served spec), https://docs.whisperr.net/api/overview/
api: Whisperr Runtime API
base_url: https://api.whisperr.net
operations:
  - previewDecision
  - getUserState
  - getUser
---

# Preview a retention decision without sending anything

`POST /v1/decisions/preview` (`previewDecision`) evaluates which intervention
Whisperr's engine would select for a user and returns it **without dispatching
a message to that person**.

This is the safest operation on the public surface, and the right first call for
an agent, an evaluation harness, or anyone testing an integration. Every other
write on this API either records data (`trackEvent`, `identifyUser`) or, further
down the pipeline, results in a real email to a real customer. This one does
neither.

## Why this matters

Whisperr publishes **no sandbox and no test mode** — there are no test keys, no
magic test identifiers, and no fixture users. There is therefore no way to
exercise the delivery path safely. `previewDecision` is the only published
dry-run affordance, so treat it as the substitute for a sandbox.

## Step 1 — Make sure the user has state

Decisioning reads accumulated state. Before previewing:

- `GET /v1/users/{external_id}` (`getUser`) — confirm the user exists.
- `GET /v1/users/{external_id}/state` (`getUserState`) — inspect the computed
  runtime state the decision will be made against.

If the user is unknown you will get a `404`. Send an `identifyUser` call and the
relevant events first.

## Step 2 — Preview

```bash
curl -X POST https://api.whisperr.net/v1/decisions/preview \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $WHISPERR_API_KEY" \
  -d '{ /* PreviewDecisionRequest */ }'
```

The response (`PreviewDecisionResponse`) describes the selected intervention and
variant. Consult the served OpenAPI at
<https://api.whisperr.net/openapi.json> for the exact request and response
shape — these two schemas are the authoritative description, and unlike the
ingestion endpoints they are not mirrored in prose documentation.

## Step 3 — Interpret the result

Remember what governs whether anything would fire at all:

- Only event types **configured during onboarding** drive interventions. Events
  outside that catalog are accepted but inert, so a preview returning no
  intervention is frequently a configuration gap rather than a bug.
- Interventions and their variants can be individually enabled or disabled from
  the dashboard, so a variant that exists may still be switched off.

## Step 4 — Handle errors

Standard envelope: `{"error":{"code","message","request_id"}}`.

- `401` / `403` — auth. Check the `wrk_` key; stop rather than retry.
- `404` — unknown user. Identify them and send events first.
- `429` / `5xx` — transient. Back off exponentially, honor `Retry-After`.
- other `4xx` — malformed request. Fix the body; unknown fields are rejected
  outright.

## Guardrail for autonomous callers

Bind an agent to `previewDecision`, `getUser` and `getUserState` when you want it
to reason about retention posture. Adding `trackEvent`/`trackEventBatch` lets it
write into the timeline that drives real customer messaging, and `identifyUser`
lets it change contact addresses and consent flags. Those are meaningfully
different consequence classes — grant them separately and deliberately.
