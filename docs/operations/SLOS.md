# SLOs, alerting and safe deployment: Direct Digital Channel

We set the targets per country over a rolling 30-day window, and we report them per calendar month against C1.

## 1. How we measure the SLIs

| Item | Rule |
|---|---|
| Where | In Application Insights, at the BFF that serves the country: `channel-bff`, and `mx-quote-bff` for the current MX journeys until they move. The country comes from the validated token (`CURRENT_STATE.md:89`) |
| Bad request | A 5xx, or a BFF timeout (4 s, `CURRENT_STATE.md:90`). INC-0907 did not count the customers who left while waiting (`INC-0907.md:28`) |
| Excluded | Invalid requests (4xx caused by customer input), synthetic checks and load tests |
| Counting | Good requests ÷ valid requests. Each bad request keeps its correlation ID, so we can group the failures by cause |
| Complements | Synthetic checks per country, for hours with little traffic and for the frontend. We use `quote-xapi` metrics only for diagnosis, because they do not see the BFF timeout |
| Purchase events | Payment confirmed (Payments) → policy issued → policy sent (Communications), joined by quote ID. We start recording them now in the current journeys, so we have a baseline before the launch |

## 2. SLOs

| Journey | SLI | Target MX · CO · CL | Why this target |
|---|---|---|---|
| Get a quote: Home Essentials | Quotes answered without error ÷ valid quote requests | 99.5% · 99.5% · 99.5% | C1. The price grid answers even when the org is down ([ADR-0002](../adr/ADR-0002-channel-to-core-integration.md)). Salesforce is needed only for missing combinations and during the 05:00–06:00 UTC load, at night in the three countries, so the exposure is at most 60 min × 30 × 3% ≈ 54 min per month |
| Get a quote: current products | Same | 98.1% · 99.2% · 98.9% | We start from today's synthetic availability, which is higher than the org's 97% (`DELIVERY_METRICS.md:17`; [assumption AS-05](../../ASSUMPTIONS.md)). Rating runs in Salesforce for every quote, even with the table cache (`quote-papi-mx.log:27-28`), so org outages reach the customer. The org is down ≈ 21.6 h per month (3% × 720 h, `SALESFORCE_CONSTRAINTS.md:15`) and C1 allows only 3.6 h (0.5% × 720 h). Each target goes up toward 99.5% when the journey adopts the grid |
| Quote latency: all products | Quotes answered in < 2.5 s ÷ valid quote requests | ≥ 95% every day | C2 (p95 < 2.5 s), evaluated per day so that month-end peaks count |
| Buy: purchase confirmed in session | Purchases confirmed in the session (payment accepted, customer told that the policy will follow) ÷ valid purchase attempts | 99.5% | Issuance is asynchronous, and when the org is down we skip the check before payment ([ADR-0002](../adr/ADR-0002-channel-to-core-integration.md), [assumption AS-03](../../ASSUMPTIONS.md)). For this reason, a Salesforce outage does not block the purchase |
| Issue: issued and sent in ≤ 3 min | Policies issued and sent ≤ 3 min after payment confirmation ÷ paid purchases | 95% (initial) | BR-01. Issuance runs in Salesforce (≈ 97%), and purchases made during an outage wait until the org is back ([assumption AS-04](../../ASSUMPTIONS.md)). We recalibrate this target after 30 days of data |
| Issue: completeness | Policies issued and sent ≤ 24 h after payment ÷ paid purchases | 99.9% | No paid purchase is lost. The remaining cases go to the exception queue for manual issuance or refund ([ADR-0002](../adr/ADR-0002-channel-to-core-integration.md)) |

The purchase targets are for Home Essentials. For the current products we set them after 30 days of measurement, because today there is no data on purchases or issuance (`DELIVERY_METRICS.md`).

## 3. Error budget policy

The budget is 1 − target over the rolling 30 days, per country and SLO. For Home Essentials, 15,000 quotes per month (BR-01) × 0.5% = 75 failed quotes.

| State | Trigger | What the cell does |
|---|---|---|
| At risk | More than 50% of the budget spent in the last 15 days | Reliability work goes first in the next sprint for that country, and releases to that country need Technical Lead approval |
| Exhausted | 100% of the budget spent | We freeze all changes in that country, including Nimbus changes, except reliability and security fixes, until the SLO is back within budget. A shared component still deploys, with its new features off in that country ([ADR-0001](../adr/ADR-0001-regional-reuse-model.md)). A postmortem is required, even if no incident was declared |
| Target review | Every quarter, with the Product Owner | After the first recalibration, targets only go up. A target goes up when the journey has met the higher value for 30 days |

## 4. Alerting

We page someone only when customers are failing now, or when a shared resource is close to affecting another cell. Everything else opens a ticket, and the CPU and memory alerts ([tech debt TD-07](../../context/TECH_DEBT_INVENTORY.md)) move to dashboards.

The paging thresholds use the 99.5% of C1 for every journey, not the interim targets. With the interim targets, the INC-0907 replay pages MX only at 17:00 and never pages CO or CL.

| Alert | Condition (per country and journey) | Action | 30 Sep replay |
|---|---|---|---|
| Fast burn | Bad ratio ≥ 14.4 × 0.5% = 7.2% over 1 h and over the last 5 min | Page | MX 15:50; CO and CL 16:40 (`channel-metrics.csv`, `quote-xapi` 5xx only) |
| Slow burn | Bad ratio ≥ 6 × 0.5% = 3% over 6 h and over the last 30 min | Page | — |
| Salesforce request rate | The channel sends more than 32 req/s to an org for 5 min, which is 80% of the agreed 40 (`SALESFORCE_CONSTRAINTS.md:19`) | Page | MX 15:15 |
| Org allocation | The org reaches 75% of its daily allocation, or the channel reaches 90% of its daily budget | Page, and notify Payments and the Salesforce platform team | MX 16:15, 40 min before the Payments pause (`shared-metrics.csv`) |
| Budget warnings | 50% or 75% of an error budget, or of the channel's Salesforce budget | Ticket | — |
| Grid load late | The Home Essentials grid is not fully loaded at 06:00 UTC | Ticket | Live rating covers the missing combinations |

For current products, a Salesforce outage also sends a page. The on-call checks that quotes fail fast, informs the contact center and joins the Salesforce platform team.

## 5. Safe deployment

Shared components run as one regional deployment, configured per country and with per-country bulkheads ([INC-0907 action A3](INC-0907-POSTMORTEM.md#5-action-items), [ADR-0001](../adr/ADR-0001-regional-reuse-model.md)).

| Stage | Rule | Source |
|---|---|---|
| Path to production | Every change, including Nimbus changes, goes through Meridian pipelines (C7). Until then, each Nimbus release to MX needs Technical Lead approval | [Actions A7a, A7b](INC-0907-POSTMORTEM.md#5-action-items) |
| Before production | A performance test with a budget of Salesforce calls per quote, which fails above the baseline + 20%, and a lint of the retry configuration | [Actions A1, A2](INC-0907-POSTMORTEM.md#5-action-items) |
| Progressive traffic shift | 5% → 25% → 100% of the traffic. Shared components go to the three countries at the same time; single-country components (such as `quote-papi-mx`) go only to their country. Each step lasts until every country has enough traffic to compare, and CL takes the longest | — |
| Observed, per country | Bad ratio and p95 at the BFF, Salesforce calls per quote, and requests per second per org | §1 |
| Automatic rollback | If any country passes the fast-burn threshold, has a p95 ≥ 2.5 s or makes more calls per quote than the baseline + 20%, we redeploy the previous version in every country | A1 |
| Flags per country | A new feature is turned on one country at a time, after Product Owner approval. A flag also turns a feature off in one country without a deployment | [TD-10](../../context/TECH_DEBT_INVENTORY.md), [`ENGINEERING_STANDARDS.md` §2](../engineering/ENGINEERING_STANDARDS.md#2-delivery-flow-and-quality-gates) |
| Freeze | The last two days of each month, and any country whose budget is exhausted | [Action A6](INC-0907-POSTMORTEM.md#5-action-items), §3 |

The cost of this choice is that a failure in one country rolls back the shared component in the three countries. With per-country flags, the other two countries can keep the new feature.
