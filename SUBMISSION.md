# Submission

| Item | Value |
|---|---|
| Name | German Bedoya Salazar |
| Candidate ID | monobedoya@gmail.com |
| Date submitted | 2026-11-23 |

## Summary

I started from the evidence. INC-0907 and PR-388 show the same problems that the target architecture must remove: one pool for the three countries, direct calls to Salesforce, no rating-table cache with immediate retries, vendor changes without gates, no SLOs, and identity taken from headers. Home Essentials is the vehicle to fix them without a big bang: we build it once on the regional platform, and Mexico joins that platform with this product instead of migrating all its journeys first ([ADR-0001](docs/adr/ADR-0001-regional-reuse-model.md)). Quotes reach Salesforce only through the API-led layers, and a price grid computed every morning keeps Home Essentials quotes available while an org is down. With orgs at ≈ 97%, this is the only way I found to reach 99.5% ([ADR-0002](docs/adr/ADR-0002-channel-to-core-integration.md)). The SLOs start from the real availability of the current products and page at 99.5% for all of them ([`SLOS.md`](docs/operations/SLOS.md)). The roadmap closes the P1 actions of the incident first, uses the Nimbus renewal of 31 Jan 2027 to bring the vendor into Meridian pipelines, and launches each country before its month-end freeze ([`ROADMAP.md`](docs/roadmap/ROADMAP.md)). For BR-02 the answer is no for customers now, a test in March and a go/no-go on 15 Jul 2027.

## Time spent

| Part | Time |
|---|---|
| Reading the material | 2 h 30 min |
| A — Architecture, standards and governance | 4 h |
| B — Quality and automation | 30 min |
| C — Reliability and operations | 2 h 45 min |
| D — Pull request review and mentoring | 30 min |
| E — Roadmap and feasibility | 40 min |
| F — Responsible AI | 45 min |
| Cross-document review and submission | ≈ 1 h |
| **Total** | **≈ 12 h 40 min** |

The total is higher than the suggested time because I tested each main decision against alternatives before the documents were written. The time of the AI tool is declared in [`AI_USAGE.md`](AI_USAGE.md), not here.

## The three decisions I am most confident about

1. **A price grid for Home Essentials ([ADR-0002](docs/adr/ADR-0002-channel-to-core-integration.md)).** Each org is available ≈ 97% of the month, so a quote that always needs Salesforce cannot reach 99.5% (C1). Prices calculated once a day are valid for 24 hours (`SALESFORCE_CONSTRAINTS.md:25`), so the dependency moves from every quote to one daily load, and the price keeps one source of truth.
2. **SLO targets from today's baseline, with alerts at 99.5% ([`SLOS.md`](docs/operations/SLOS.md)).** A 99.5% target that the current products can never meet would be ignored. The targets go up when each journey improves, but the alerts still use 99.5%: in the INC-0907 replay, MX pages at 15:50 instead of 17:00.
3. **One regional deployment with bulkheads per country ([ADR-0001](docs/adr/ADR-0001-regional-reuse-model.md)).** CO and CL already run this pattern, and MX joins it. In INC-0907, CO and CL failed because the shared pool had no limit per country and the timeout was 30 s, not because the deployment was regional. Bulkheads, short timeouts and flags per country remove that cause at the lowest cost.

## The decision I am least sure about

**The target of 95% of policies issued and sent in ≤ 3 minutes ([`SLOS.md`](docs/operations/SLOS.md) §2).** BR-01 asks for every policy in under three minutes. However, issuance runs in Salesforce (≈ 97%), and today there is no data on purchases or issuance (`DELIVERY_METRICS.md`), so 95% is an estimate ([AS-04](ASSUMPTIONS.md)). For this reason we start recording the events from payment confirmed to policy sent in the current journeys now. What would change my mind: 30 days of those events. If the measured share is lower, the sponsor must choose between a lower target and more investment in issuance.

## What I would do with more time

- Run the December spike: real grid size per country, load time and the bands for the five answers ([AS-01, AS-02 and AS-07](ASSUMPTIONS.md)).
- Put numbers on the cost profile (C9) of the regional platform, the grid store and the DR replicas, not only the comparison between options.
- Write the AsyncAPI contract of the payment-confirmed event and the detailed flow of the issuance exception queue, with Payments and Communications.
- Replay the alerts and the rollback rules on more month-ends than 30 Sep, to tune the thresholds.
- Design the token exchange (RFC 8693) with the Identity team, so `quote-xapi` does not depend on user context headers built by the BFF.
