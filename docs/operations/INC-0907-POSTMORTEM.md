# INC-0907 Postmortem: quote journey degraded in MX, CO and CL

| Item | Value |
|---|---|
| Date | 30 Sep 2026, last day of the month (`INC-0907.md:5`) |
| Customer impact | 15:25–17:20 UTC (1 h 55 min) in the three countries; MX payment notifications delayed 16:55–18:20 |
| Severity | Declared SEV3. Should have been SEV2 at 15:46, and SEV1 once MX payments stopped (§1) |

**Summary** (blameless: about the system and the process, not about people). `quote-papi-mx` 3.8.0, deployed the night before month-end, turned each MX quote into ≈ 8 Salesforce calls instead of ≈ 1.3 and retried timeouts immediately. At the MX month-end peak, the channel pushed the MX org past its 40 req/s limit. Salesforce slowed down, and slow MX requests filled the in-flight pool of `quote-xapi`, which the three countries share, so CO and CL were rejected too. The MX org reached 85% of its daily allocation and Payments paused MX consumers. Rolling back to 3.7.2 mitigated the incident.

## 1. Triage as it should have been done

| Item | What should have happened |
|---|---|
| Severity | **SEV2 at 15:46**, when the on-call engineer saw errors in the three countries on the core journey: "major feature degraded for many users" (`INC-0907.md:13`; chat 15:46). **SEV1 if MX payments stop** ("service down for all users" of payments in MX), which happened at 16:55. With the involvement below, that SEV1 was avoidable |
| Impact at 15:45 | MX: 22.6% 5xx at `quote-xapi`, p95 30 s (the cap). CO: 8.3%, p95 15.2 s. CL: 8.3%, p95 16.4 s (`channel-metrics.csv 15:45Z`). The BFF gives up at 4 s (`CURRENT_STATE.md:90`), so customers saw more failures than these 5xx |
| Total impact | Requests failed at `quote-xapi`: MX ≈ 9,400 · CO ≈ 1,500 · CL ≈ 700. Contact center calls: 1,140 · 860 · 310 (`INC-0907.md:22-26`). Customers who gave up without an error are not counted (`INC-0907.md:28`) |
| Involve from 15:46 | Incident commander; channel on-call; integration engineer (`quote-xapi`); Nimbus on-call (`quote-papi-mx` held 61–62 of 64 slots per replica, `quote-xapi.log:12-13`, and was the latest change, `TIMELINE.md:8`); contact center leads of the three countries; Director of Digital Channels; Identity team, to postpone the 16:00 key rotation |
| Involve by 16:15 at the latest | Salesforce platform team and Payments Integration Technical Lead. The MX org was at 75.6% and rising ≈ 1.1 points every 5 min (`shared-metrics.csv 15:45Z–16:15Z`), so the 85% payments pause (`SALESFORCE_CONSTRAINTS.md:37`) was about 40 min away |

**First communication (≈ 15:55):**

> SEV2. The quote journey is degraded in MX, CO and CL since about 15:25 UTC: customers get errors or wait more than 4 s, and Mexico is the most affected. Current lead: `quote-xapi` is saturated by slow responses from `quote-papi-mx` (3.8.0, deployed 29 Sep 23:40). All non-essential changes are on hold, including the 16:00 key rotation. Incident commander: Technical Lead, Direct Digital Channel. Next update at 16:15 UTC.

## 2. Timeline (UTC)

| Time | Event | Evidence |
|---|---|---|
| 09-29 23:40 | `quote-papi-mx` 3.8.0 deployed from Nimbus CI: rating-table cache off, 4 attempts, no backoff. Release notes: "No configuration changes required" | `TIMELINE.md:8,20`; `quote-papi-mx.log:2` |
| 14:00 | Before any impact, MX quotes cost 8.3 Salesforce calls each (≈ 1.3 on normal days) and the MX org is at 59.0% (normal daily peak 45.8–48.6%). No alert exists | `channel-metrics.csv 14:00Z` (5,021 ÷ 605); `shared-metrics.csv 14:00Z`; `sf-api-usage-daily.csv` |
| 15:00 | MX month-end peak starts (3.3× by 16:20). Funnel report starts on `pg-channel-co` | `channel-metrics.csv`; `pg-channel-co.log:2` |
| 15:15 | Channel exceeds 40 req/s on the MX org (44.0); Salesforce p95 1.5 s | `channel-metrics.csv 15:15Z` |
| 15:27 | Salesforce p95 passes the 5 s timeout; immediate retries; first MX `504` after 30 s | `channel-metrics.csv 15:25Z–15:30Z`; `quote-xapi.log:6` |
| 15:34–15:38 | `quote-xapi` in-flight limit reached; CO queued; first `503 QUEUE_FULL` for CL and CO | `quote-xapi.log:7,10-11` |
| 15:41–15:46 | CO contact center reports failures; `quote-xapi` logs MX at 61/64 and 62/64 in-flight slots; on-call sees errors in CO, CL and MX, CPU 38% | chat 15:41, 15:46; `quote-xapi.log:12-13` |
| 15:52 | Declared SEV3; CO assumed to be the most affected | chat 15:52 |
| 16:00–16:11 | Key rotation causes 401s until the JWKS refresh (16:05); database hypothesis raised and ruled out | `quote-xapi.log:16-20`; chat 16:02–16:11 |
| 16:14–16:21 | Vendor: 3.8.0 "should help, not hurt"; `quote-xapi` scaled from 2 to 4 replicas | chat 16:14, 16:18; `quote-xapi.log:22-23` |
| 16:20–16:43 | MX Salesforce rate 93.7–95.7 req/s, `quote-papi-mx` p95 46–52 s; CO and CL improve, then `quote-xapi` saturates again (256/256) | `channel-metrics.csv`; `shared-metrics.csv`; chat 16:31, 16:43 |
| 16:55 | MX org at 85%: Payments pauses MX consumers | chat 16:55 |
| 16:58 | Salesforce platform team: channel user at ≈ 856k calls since 00:00 (budget 250k), ≈ 90 req/s, mostly `Rating_Table__c` | chat 16:58 |
| 17:00–17:20 | Rollback to 3.7.2 (warm cache, 1 call per quote); mitigated in the three countries | chat 17:00–17:20; `quote-papi-mx.log:24-27` |
| 18:20 | Temporary +500k calls granted; payments resume; incident closed | chat 18:20 |

## 3. Root cause, contributing factors and ruled-out hypotheses

**Root cause.** 3.8.0 removed the rating-table cache and retried Salesforce timeouts immediately, up to 4 attempts (`quote-papi-mx.log:2`). Salesforce calls per MX quote rose from ≈ 1.3 to 8.3, and to 15.0 with retries (`channel-metrics.csv 16:30Z`: 28,577 ÷ 1,906). At the 3.3× month-end peak, the channel sent 44–96 req/s to the MX org against the agreed 40 (`SALESFORCE_CONSTRAINTS.md:19`), and each timeout added more load. Traffic alone was not the cause: at 17:20, on 3.7.2, MX handled 92% of peak traffic at p95 831 ms (`channel-metrics.csv 17:20Z`: 1,831 ÷ 1,987).

| # | Contributing factor | Evidence |
|---|---|---|
| CF1 | One in-flight pool and one queue in `quote-xapi` for the three countries; MX held 91–96% of the slots | `CURRENT_STATE.md:74-75`; `shared-metrics.csv 15:35Z–17:00Z` |
| CF2 | `quote-xapi` waits 30 s with no circuit breaker, while the BFF gives up at 4 s. By Little's law MX needed ≈ 190–198 slots (MX req/s × 30 s) against 128 | `CURRENT_STATE.md:76,90`; `channel-metrics.csv 15:35Z` (1,901 ÷ 300 × 30) |
| CF3 | Retries on timeout with no backoff, no jitter and no retry budget | `quote-papi-mx.log:2,10-22` |
| CF4 | Cache removed to "always use fresh prices", although business accepts tables up to 24 h old if the version is recorded. The configuration changed, yet the notes said it did not | `TIMELINE.md:17,20`; `SALESFORCE_CONSTRAINTS.md:25` |
| CF5 | `quote-papi-mx` calls the MX org directly, not through `sf-policy-sapi-mx` and the API Manager: no per-consumer rate limit or visibility (C4, C6) | `CURRENT_STATE.md:66-67` |
| CF6 | No consumption alerts per integration user and no SLO alerts: the overrun was visible from the morning and nobody saw it | `SALESFORCE_CONSTRAINTS.md:38`; `shared-metrics.csv 14:00Z` |
| CF7 | Nimbus deploys from its own CI with no Meridian gates and no performance tests, here on the eve of month-end | `CURRENT_STATE.md:101`; `DELIVERY_METRICS.md:13`; `TIMELINE.md:8` |
| CF8 | A planned change (key rotation) on the riskiest day of the month | `TIMELINE.md:10` |
| CF9 | Response: low severity, anchoring on the first reporting country, scale-out without a hypothesis, Payments informed only at 85% | chat 15:52, 16:18, 16:55 |

| Ruled-out hypothesis | Why | Evidence |
|---|---|---|
| `pg-channel-co` overloaded by the funnel report | CO failed at `quote-xapi` (`QUEUE_FULL`), before reaching the database; BFF queries took 37–44 ms; errors continued after the report ended (15:58) | `quote-xapi.log:11`; `pg-channel-co.log:5-8`; chat 16:09, 16:11 |
| Key rotation | 401s only from 16:00:03 to 16:05:02, 35 min after the degradation began | `quote-xapi.log:16-20` |
| `channel-bff` 5.2.0 | Deployed 44 h earlier; the CO and CL Process APIs stayed healthy (p95 616–689 ms) | `TIMELINE.md:7`; `channel-metrics.csv` |
| `quote-xapi` short of CPU or memory | CPU 38%; the limit hit was the in-flight cap | chat 15:46; `quote-xapi.log:7` |
| Salesforce platform outage | CO and CL orgs normal (p95 ≈ 1.0–1.1 s); MX latency followed the channel's own request rate | `channel-metrics.csv` |

## 4. Five whys and response analysis

**Why did MX fail?** (1) `quote-papi-mx` took 30–50 s per quote. (2) Salesforce queries timed out at 5 s and were retried at once. (3) Salesforce slowed under up to 96 req/s against 40. (4) 3.8.0 made each quote ≈ 8 calls (≈ 15 with retries) at triple traffic. (5) The change shipped with no performance test, no Salesforce-consumption check and an unreconciled pricing request (CF4, CF7).

**Why did CO and CL fail with healthy back ends?** (1) `quote-xapi` queued or rejected them. (2) MX held 91–96% of the slots. (3) Each MX request held a slot for up to 30 s. (4) One pool and one queue for all countries, no circuit breaker. (5) `quote-xapi` became regional in 2026-Q2 for reuse (C3) with no isolation requirement (C1) and no resilience standard (`CURRENT_STATE.md:65`).

**Why almost two hours?** (1) The cause was found at 16:58, 1 h 06 min after the declaration. (2) The team followed the first reporting country and two coincident events, and scaled out instead of reading per-route saturation. (3) No alert pointed at the symptom or the cause, and SEV3 brought in neither the vendor, the Salesforce platform team nor Payments. (4) No "what changed in 72 h?" step; the vendor's view was accepted without data. (5) No shared incident playbook.

| Helped | Made it worse |
|---|---|
| CO contact center reported quickly (15:41) | SEV3, and CO assumed to be the origin (15:52) |
| The DBA ruled out the database with evidence in 7 min (16:02–16:09) | The scale-out raised Salesforce load from 75 to 96 req/s and the org from 75.6% to 85.8%, which led to the payments pause |
| The Identity team reported the rotation and its effect openly (16:06) | The key rotation went ahead during an active incident |
| The Salesforce platform team gave per-integration consumption on request (16:58): the decisive clue | The vendor's "should help" was accepted without checking calls per quote or the release configuration |
| Rollback decided and executed in 5 min, with a warm cache | The per-route log of `quote-xapi` (15:44) went unused; there are no per-country dashboards |

## 5. Action items

Priority: **P1** before the next month-end peak · **P2** within the quarter · **P3** roadmap.

| # | Action | Owner | Priority | How we verify it works |
|---|---|---|---|---|
| A1 | Rating-table cache mandatory in every quote Process API, refreshed after the 05:00 UTC publication; each quote records the table version | Nimbus (`quote-papi-mx`); Technical Lead (standard) | P1 | Salesforce calls per quote ≤ 1.5 per org on the dashboard; a performance-test gate fails above baseline + 20% |
| A2 | Retry standard: no retry on timeout; at most 1 retry on fast failures (connection error, `503`) with backoff and jitter; retry budget ≤ 10%; total deadline below the caller's timeout | Technical Lead; all teams | P1 | Configuration lint in the pipeline; fault injection (Salesforce at 6 s) keeps calls per quote ≤ 2 |
| A3 | Per-country bulkheads in `quote-xapi` (in-flight limit and queue per country), a circuit breaker per route, and an outbound timeout aligned to the BFF's 4 s | Integration engineer | P1 | Load test with 30 s latency on the MX route: CO and CL p95 < 2.5 s and zero `QUEUE_FULL` |
| A4 | Salesforce consumption alerts per integration user and org: > 32 req/s (80% of 40) for 5 min; 24-h budget at 50/75/90%; org ≥ 75% notifies the channel and Payments | Salesforce platform team; Technical Lead | P1 | Replay of 30 Sep: budget alert hours before the impact, rate alert by 15:15, org alert at 16:15 |
| A5 | SLO alerting per country and journey, measured at `channel-bff` with timeouts included (see `SLOS.md`) | Technical Lead | P1 | Replay of 30 Sep pages for MX by 15:50 at the latest |
| A6 | Change freeze in the last two days of each month for channel components and shared dependencies, key rotations included | Technical Lead; change management; Identity team | P1 | Pipelines block production deploys in the window; the change calendar shows no month-end changes |
| A7a | Interim: every Nimbus production release in MX needs Technical Lead approval and declares configuration changes and Salesforce calls per quote | Technical Lead; Nimbus | P1 | No MX release without a recorded approval and declaration |
| A7b | Nimbus components deploy through Meridian pipelines with the same gates (performance test with a Salesforce-call budget, canary with automatic rollback); Nimbus's deployment credential is revoked; both become terms of the 31 Jan 2027 renewal (C8) | Technical Lead; Nimbus; vendor management | P2 | 100% of MX production deploys come from Meridian pipelines |
| A8 | `quote-papi-mx` reaches Salesforce through `sf-policy-sapi-mx`, with per-consumer rate limiting in the API Manager (C4) | Nimbus; Salesforce platform team | P2 | Zero direct calls from the channel to the MX org (integration-user login history) |
| A9 | Incident playbook: severity examples (multi-country quote degradation is at least SEV2; payments stopped is SEV1), incident commander rotation, first-15-minutes checklist ("what changed in 72 h?", per-route saturation, shared-dependency usage, check downstream before scaling) | Director of Digital Channels; Technical Lead | P2 | Game day before the next peak: declared within 15 min at the right severity |
| A10 | Protocol with Payments and the Salesforce platform team: at org ≥ 75% the channel throttles first (kill switch), before Payments pauses; pre-agreed emergency allocation | Technical Lead; Payments TL; Salesforce platform team | P2 | Tabletop exercise; protocol recorded in `INTERFACES_AND_GOVERNANCE.md` |
| A11 | Reconcile "always fresh prices" with the 24-h rule: quotes show the table version, and any change to caching needs Product Owner sign-off | Product Owner; MX pricing team | P1 | Decision recorded and linked from the caching standard (A1) |
| — | **Latent risks** (did not cause INC-0907): move the month-end funnel report off the primary database; publish IdP keys in JWKS before use and refresh on unknown `kid` | Data & Analytics (report); Identity team (keys) | P3 | No reporting sessions on the primary; next rotation with zero 401s |
