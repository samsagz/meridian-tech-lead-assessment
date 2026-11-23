# ADR-0002: Channel-to-core integration

| Item | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-22 |
| Deciders | Technical Lead (Direct Digital Channel), Salesforce platform team lead, Payments Integration lead, Architecture office |
| Related | C1, C2, C4, C6; [ADR-0001](ADR-0001-regional-reuse-model.md); [`SLOS.md`](../operations/SLOS.md); [INC-0907 actions](../operations/INC-0907-POSTMORTEM.md#5-action-items) A1 (rating-table cache), A2 (retry standard) and A8 (`quote-papi-mx` through the System API); [assumptions](../../ASSUMPTIONS.md) AS-01 to AS-03 (price grid and issuance) and AS-07 (grid store) |

## Context

- Today `quote-papi-mx` and the issuance in `mx-quote-bff` call the MX org directly, against C4 (`CURRENT_STATE.md:64,66`).
- Each org is available ≈ 97% of the month (`SALESFORCE_CONSTRAINTS.md:15`): ≈ 21.6 h per month without the org (3% × 720 h), while C1 allows 3.6 h (0.5% × 720 h). If every quote needs the org, the quote journey cannot reach 99.5%.
- Rating p95 is 1.1 s and issuance p95 is 4 s (`:16,18`).
- The channel has a daily budget per org (MX 250,000 calls) and must stay below 40 req/s (`:19,31`). With the table cache, a quote costs ≈ 1.3 calls; on 30 Sep, without it, MX made 6.24 calls per quote (`sf-api-usage-daily.csv:20`).
- Tables are published at 05:00 UTC, and prices from tables up to 24 h old are valid if the quote shows the version (`SALESFORCE_CONSTRAINTS.md:24-25`).

## Decision drivers

1. Quote availability per country when the org is down (C1).
2. Salesforce consumption inside the budget and the rate limit, also at month-end (C6).
3. Latency (C2).
4. Compliance with C4, without exceptions.
5. Effort that fits one MuleSoft developer (`TEAM.md:19`).

## Options considered

| Dimension | A — API-led, table cache, live rating | B — A plus a price grid for Home Essentials and asynchronous issuance | C — Direct access under a C4 exception |
|---|---|---|---|
| Complexity | Low: the CO and CL pattern (`CURRENT_STATE.md:81`) | Medium: daily grid load, versions, issuance queue | Low now; high to revert later |
| Cost | Base | Grid storage and a daily batch in Salesforce | Lowest at first |
| Availability and isolation | Quotes follow the org (≈ 97%) | Home quotes need the org only for missing combinations and during the 05:00–06:00 UTC load ([`SLOS.md` §2](../operations/SLOS.md#2-slos)) | Same as A, without rate limiting per consumer (INC-0907) |
| Salesforce consumption | ≈ 1.3 calls per quote. With the 24–28 Sep rate, MX on 30 Sep would make 157,745 × 1.31 ≈ 207,000 calls: 83% of its budget (`sf-api-usage-daily.csv`) | Home: a few bulk calls per day (`SALESFORCE_CONSTRAINTS.md:35`), almost none per quote | No control per consumer |
| Latency | One rating call (p95 1.1 s) | A grid lookup, without Salesforce | Same as A |
| Security and privacy | Credentials per consumer at the System API | Same as A | Salesforce credentials in the channel; needs an ADR and Security approval |
| Fit with the team and the vendor | Known by the internal team | Needs the Salesforce platform team for the grid (AS-02) | What Nimbus does today |

Also discarded: asynchronous quoting, because BR-01 asks for the purchase in the same session; and rating inside the channel, because it creates a second source of truth for the price.

## Decision

Option B for Home Essentials. Option A for the current products until they adopt the grid.

| Flow | Rule |
|---|---|
| Path | BFF → `quote-xapi` → consolidated quote Process API → `sf-policy-sapi-{country}` → org. `quote-papi-mx` moves to `sf-policy-sapi-mx` (A8), and `mx-quote-bff` issues policies through this path |
| Table cache | In the Process API, in memory, loaded at start-up and after 05:00 UTC; each quote records the version (A1). Not in the System API, which belongs to the Salesforce platform team and must stay thin |
| Home price grid | Computed in Salesforce after 05:00 UTC, exported by the System API in bulk calls (AS-01, AS-02) and kept in a store next to the Process API (AS-07). After 05:00 we do not serve the previous version. The grid loads in segments, the most frequent first (the capital city when there is no history), with a target of 06:00 UTC |
| Live rating | For missing combinations and current products: one attempt with a ≈ 1.2 s timeout, no retry on timeout, at most one retry on fast failures, and a circuit breaker per country (A2). A retry after a timeout does not fit in C2: 1.2 s + 1.1 s = 2.3 s before the other hops. For Home Essentials, the live price is written into the grid |
| Org down, no price | Fail fast, save the draft and offer to resume through Communications, with a legal basis or consent (C5) |
| Purchase and issuance | Payment in the session. While the org is up, we validate the issuance data before payment (AS-03); when it is down, we skip this check. Issuance is asynchronous, with a queue per country, an idempotency key and backoff ([`ARCHITECTURE.md` §2](../architecture/ARCHITECTURE.md#2-containers-c4-level-2)). A case not issued in 24 h goes to an exception queue with an owner, for manual issuance or refund |
| Consumption control | Rate limit per consumer and country at each System API, below 40 req/s; alerts in [`SLOS.md` §4](../operations/SLOS.md#4-alerting) |

If the Salesforce platform team cannot commit to the grid before 15 Dec 2026, plan B of AS-02 applies. If the cell lacks capacity, we drop the load order first, and second we stop writing live prices into the grid.

## Trade-off analysis

A is simpler, but quotes stay at the org's ≈ 97%. B moves the dependency on Salesforce from every quote to one daily load, and the 24-hour rule makes this valid. Rating stays in Salesforce, so the price has one source of truth. C saves one hop, but it repeats a cause of INC-0907: no visibility and no rate limit per consumer.

## Consequences

- **Becomes easier:** quoting Home during an org outage; C6 at month-end; one integration path for the three countries.
- **Becomes harder:** a Q1 dependency on the Salesforce platform team; the grid needs versions, verification and an alert when it is late; the customer sees more states (issuance pending, refund); one more hop than direct access.
- **To revisit, and when:** after the December spike (grid size and load time); after 30 days of Home data (grid hit rate, issuance in ≤ 3 min); when a current product adopts the grid.

## Action items

1. [ ] Ask the Salesforce platform team for the grid commitment and the MuleSoft loan before 15 Dec 2026.
2. [ ] December spike: grid size per country and load time.
3. [ ] Payment-event contract with Payments and a sending contract with Communications.
4. [ ] Move `quote-papi-mx` and the `mx-quote-bff` issuance to the API-led path (A8).
5. [ ] Salesforce calls per quote in the performance gate (A1).
