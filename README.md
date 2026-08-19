# Whisperr, Inc.

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Whisperr, Inc. builds an autonomous customer-retention platform: it ingests product events ("churn
signals"), decides on an intervention, generates the content, and delivers it.

It publishes a real developer surface. An earlier round of this profile recorded "no public developer
API" — that was wrong, and this profile corrects it.

## What Whisperr publishes

- **OpenAPI 3.0.3** served at <https://api.whisperr.net/openapi.json> — 46 paths, 53 operations,
  64 schemas. Six of those operations are public and API-key authenticated; the rest are dashboard
  operations behind a console session.
- **Developer docs** at <https://docs.whisperr.net/> covering the three ingestion endpoints and ten SDKs.
- **Ten first-party SDKs** — JavaScript/browser, React, Next.js, Node, React Native, Python, PHP,
  .NET, Flutter and Swift — plus `npx @whisperr/wizard`, an agentic integration CLI.
- **A public wire contract**, [WhisperrAI/whisperr-spec](https://github.com/WhisperrAI/whisperr-spec),
  whose executable conformance fixtures every official SDK runs in CI. Third-party clients can verify
  against the same fixtures.
- **Per-event idempotency** via `context.$message_id`, with a documented and fixture-pinned
  retain-vs-drop retry contract.
- A live unauthenticated health endpoint at <https://api.whisperr.net/health>.

## What it does not publish

No MCP server, A2A agent card, GraphQL, AsyncAPI or customer-facing webhooks; no status page,
changelog, deprecation policy or SLA; no pricing or plans; no sandbox or test credentials; no
security.txt, trust center or compliance certifications; and no `/.well-known/` documents on any host.

Backed by: 500-global — <https://whisperr.net> (the earlier `whisperr.online` address 301s here).
