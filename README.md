# Figment

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

Figment is an institutional staking infrastructure provider that runs validators and staking services
across 40+ proof-of-stake networks for asset managers, exchanges, custodians, wallets, banks and
protocol foundations.

Its public developer surface is a single REST API at `https://api.figment.io` — Figment publishes an
OpenAPI 3.1.0 document for it at <https://api.figment.io/openapi/figment-api.yaml> (126 paths, 129
operations, 55 schemas). The API abstracts network-specific staking mechanics behind one contract:
build ready-to-sign staking, delegation, undelegation, withdrawal, exit, compound and consolidation
transactions; broadcast the signed payloads; and read back validators, stakes, activities, balances,
rewards, reward rates, statements and portfolio data. Figment never holds customer keys — writes
return an unsigned transaction the customer signs in its own custody.

Coverage includes Ethereum (including Pectra 0x02 compounding validators and Figment Validator
Vaults), Solana, Cardano, Cosmos, Osmosis, Injective, NEAR, Polkadot, Polygon, Avalanche, Sui, Aptos,
Vaulta, OpenTrade stablecoin yield vaults, and an x402 payment facilitator. Figment also publishes
Elements (`@figmentio/elements`), a React component library for embeddable staking widgets, and
serves an OAuth-protected MCP endpoint at <https://docs.figment.io/mcp>.

- Website: <https://www.figment.io/>
- Documentation: <https://docs.figment.io/>
- Status: <https://status.figment.io/>
- Trust center: <https://trust.figment.io/>
- Secondary-market listing this profile was harvested from: <https://forgeglobal.com/figment_stock/>
