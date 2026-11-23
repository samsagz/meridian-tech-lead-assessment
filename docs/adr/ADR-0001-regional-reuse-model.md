# ADR-0001: Regional reuse model

| Item | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-22 |
| Deciders | Technical Lead (Direct Digital Channel), Director of Digital Channels, Architecture office, Security office, Legal |
| Related | C1, C3, C5, C8, C9; [ADR-0002](ADR-0002-channel-to-core-integration.md); [`SLOS.md` §5](../operations/SLOS.md#5-safe-deployment); [INC-0907 actions](../operations/INC-0907-POSTMORTEM.md#5-action-items) A3 (bulkheads in `quote-xapi`) and A7b (Nimbus on Meridian pipelines); [assumption AS-06](../../ASSUMPTIONS.md) (legal basis for regional services) |

## Context

- The sponsor asks to build Home Essentials once for the three countries (`BUSINESS_REQUESTS.md:17`). The team estimates 45–55 person-weeks to build it once and 75–90 to build it in each current stack (`:23-24`).
- CO and CL already share `web-quote-ui` and `channel-bff`, and `quote-xapi` is one deployment for the three countries (`CURRENT_STATE.md:60,63,65`). MX has its own stack, built by Nimbus.
- In INC-0907, MX took 61–62 of the 64 slots of each `quote-xapi` replica (`quote-xapi.log:12-13`), and CO and CL received 503. The countries shared one in-flight limit and one queue, with a 30 s timeout and no circuit breaker (`CURRENT_STATE.md:75-76`).

## Decision drivers

1. A failure or overload in one country stays in that country (C1).
2. One codebase, with the country differences in configuration (C3).
3. Personal data separated by country (C5).
4. Cost and operation that a cell of 6 developers can sustain (C9, `TEAM.md:9`).
5. A clear place for Nimbus (C8).

## Options considered

| Dimension | A — One regional deployment, with country configuration and bulkheads | B — One deployment per country from the same code | C — One stack per country (today's MX model) |
|---|---|---|---|
| Complexity | One deployment per component; the isolation must be configured and tested | Three deployments per release; versions can differ between countries | Three codebases |
| Cost | Lowest: one pool sized for the sum of the peaks | Minimum footprint × 3, mainly MuleSoft capacity | Highest in people: each feature three times |
| Availability and isolation | Bulkheads per country inside each component. A bad release reaches the three countries; a progressive traffic shift with automatic rollback contains it | Full isolation by construction | Full isolation |
| Security and privacy | One process serves three countries; data and secrets stay per country; needs a documented basis (AS-06) | Separation by construction | Separation by construction |
| Fit with the team and the vendor | The pattern the team runs today for CO and CL. A Nimbus change to shared code reaches CO and CL | Nimbus can be limited to the MX deployment | Status quo; does not meet BR-01 |

## Decision

Option A. MX joins the pattern that CO and CL already use.

| Item | Regional (one deployment) | Per country |
|---|---|---|
| Components | `web-quote-ui`, `advisor-portal`, `channel-bff`, `quote-xapi`, consolidated quote Process API, `issuance-worker`, CI/CD, observability | `sf-policy-sapi-{country}` and the org (Salesforce platform team) |
| Data and secrets | — | PostgreSQL database, as today (`CURRENT_STATE.md:68-69`); Key Vault; API Manager client ID |
| Country differences | Configuration per country, versioned and tested per country; feature flags per country ([tech debt TD-10](../../context/TECH_DEBT_INVENTORY.md)) | Country code only behind an extension point, with its own tests ([`ARCHITECTURE.md` §3](../architecture/ARCHITECTURE.md#3-regional-and-country-specific-capabilities-c3)) |

Bulkheads in each shared component:
- In-flight limit and queue per country in `quote-xapi` (A3), and a circuit breaker per country and route.
- Client ID per country with its own rate limit in the API Manager, so we can limit or revoke one country alone.
- Connection pool per country database.
- Against a bad release: code review, tests for each country configuration, progressive traffic shift 5% → 25% → 100% with metrics per country, and automatic rollback ([`SLOS.md` §5](../operations/SLOS.md#5-safe-deployment)).

Home Essentials runs on this platform in MX from day 1. The current MX journeys stay on the Nimbus stack with two minimum changes: `quote-papi-mx` and the issuance of `mx-quote-bff` move to the System API ([ADR-0002](ADR-0002-channel-to-core-integration.md), [action A8](../operations/INC-0907-POSTMORTEM.md#5-action-items)). Then they move one journey at a time.

## Trade-off analysis

B gives the strongest isolation, but it triples the minimum footprint and the release work for about 500 quotes per day per country (15,000 ÷ 30, `BUSINESS_REQUESTS.md:15`). In INC-0907, CO and CL failed because the shared pool had no limit per country and the timeout was 30 s; bulkheads and shorter timeouts remove that cause. Bulkheads do not cover a bad release or a resource problem in the shared process; the progressive traffic shift, the rollback and the per-country flags cover it. We chose A because the team already operates it, it costs the least, and the code is the same in A and B, so a component can move to B later without a rewrite.

## Consequences

- **Becomes easier:** one release reaches the three countries; a new product is mainly configuration and screens; standards apply in one place.
- **Becomes harder:**
  - Isolation depends on configuration. Every change to timeouts, retries or concurrency needs a fault-injection test: Salesforce slow in one country, and the other two keep p95 < 2.5 s.
  - A rollback affects the three countries.
  - A country with an exhausted error budget cannot stop a shared release; the new features stay off there with flags ([`SLOS.md` §3](../operations/SLOS.md#3-error-budget-policy)).
  - Capacity is planned for the three month-end peaks together.
  - Nimbus changes to shared code need a Meridian reviewer and the Meridian pipeline (A7b).
- **To revisit, and when:** if a test or an incident shows one country affecting another, that component moves to B; if Legal rejects AS-06; after the CL launch, in July 2027.

## Action items

1. [ ] Bulkheads per country in `quote-xapi` (A3).
2. [ ] Consolidated quote Process API with country configuration, Home Essentials first ([TD-01](../../context/TECH_DEBT_INVENTORY.md)).
3. [ ] MX in `channel-bff` and `web-quote-ui` for Home Essentials ([TD-03, TD-04](../../context/TECH_DEBT_INVENTORY.md)).
4. [ ] Fault-injection gate in UAT for changes to timeouts, retries or concurrency.
5. [ ] Documented basis for regional services, with Legal and Security (AS-06).
6. [ ] Rules for Nimbus on shared code in the renewal ([`ENGINEERING_STANDARDS.md` §4](../engineering/ENGINEERING_STANDARDS.md#4-nimbus-digital-c8)).
