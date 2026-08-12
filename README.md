# Delos

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

Delos Living LLC is a New York-based wellness real estate and technology company. It founded the WELL
Building Standard and co-founded the Well Living Lab with Mayo Clinic, and its connected product line —
WellCube — pairs localized air purifiers with multi-sensor devices for commercial offices.

## The API surface

Delos does not advertise a developer program, and there is no `/developers` or `/api` page on either
`delos.com` or `wellcube.io`. A real, publicly readable **OpenAPI 3.0.0 contract** does exist:

- **WellCube Cloud BE API** — <https://cloud.wellcube.io/api/v1> — 33 paths, 39 operations, served
  through a live Swagger UI at <https://cloud.wellcube.io/api/v1/docs/>.

It was found by reading <https://app.wellcube.io/config.js>, the WellCube web application's public
runtime configuration, which names the backend host. Nothing here required credentials.

Three properties of that contract are worth stating up front, because they change how it must be
consumed and all three were verified rather than assumed:

1. **Every response is HTTP 200, including failures** — confirmed live on 2026-08-12. `body.status`
   (`1` success / `0` failure) is the only outcome signal the API emits.
2. **The document declares no HTTP status codes at all.** All 19 failure responses use non-standard
   `x-`-prefixed response-map keys, which OpenAPI 3.0 does not permit. The usable error identity is
   one level down, in 14 reusable components with pinned `error.code` strings.
3. **There is no idempotency key**, on any of the 18 write operations, and no rate-limit headers are
   returned.

Delos publishes no client SDK, no changelog, no status page, no pricing, no `/.well-known/` document
and no event contract — though a live WebSocket gateway (`wss://ugw.wellcube.io/clients-socket`) is
running behind the product. Each of those absences is recorded with its probe evidence in the
corresponding artifact directory.

### Links

- <https://delos.com/> — company
- <https://wellcube.io/> — WellCube product
- <https://cloud.wellcube.io/api/v1/docs/> — the entire developer surface
- <https://github.com/Delos-tech> — GitHub organization ("Delos Living")
- <https://www.hiive.com/securities/delos-stock> — secondary-market listing that surfaced this company
