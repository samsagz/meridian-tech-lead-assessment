# Feasibility of BR-02: GenAI assistant in the quote journey

**Recommendation.** No for customers in Q1 2027. Yes to a test: a spike in March, an internal pilot with the MX contact center in May and June, and a go/no-go on **15 Jul 2027**. With a go, MX customers can ask about coverages in **Q4 2027** (phase 1). Pre-fill comes later (phase 2), when a specialized decision model is approved for our use. The service must cost less than **USD 100,000 per year** to run; we aim for **USD 15,000 or less**.

## 1. Value, effort and risk

| | Evidence | Missing information |
|---|---|---|
| Value | Coverage calls: 18,000 per month × USD 4.10 = **≈ USD 886,000 per year** (`BUSINESS_REQUESTS.md:40,43`). 41% of the customers who abandoned did not understand the coverages (`:42`) | Calls the assistant avoids; quote starts and conversion. The 41% is self-reported (≈ 2,300 answers): it shows the problem, not the size of the gain |
| Effort | Spike 2–3 pw, pilot 4–6 pw, phase 1 6–10 pw, 2–3 pw per extra country ([assumption AS-12](../../ASSUMPTIONS.md)); operation ≈ 10 pw per year | Quality of the PDFs (scanned pages, tables) |
| Cost to run | USD 3,000–14,000 per year with the small model ([AS-13](../../ASSUMPTIONS.md)) | Real token use and prices |
| Risk | Recommending can be regulated advice (`:48`); approval only for internal pilots (`:47`); documents change four times a year (`:49`); personal data in conversations (C5) | Compliance's line in each country; Security's requirements for customer use, and their lead time |

## 2. Options

| Option | Decision |
|---|---|
| A. Not now, and nothing else | Rejected: no date for the sponsor, and we learn nothing |
| **B. Spike, internal pilot, two phases** | **Recommended:** uses the existing approval and limits cost and risk |
| C. Customer assistant in MX in Q1 | Rejected: no approval, no evidence that the advice risk is controlled, and Q1 is at 77–97% of capacity ([`ROADMAP.md` §3](ROADMAP.md#3-capacity)) |
| D. Plain-language coverage summaries | **A complement** owned by Product; they are also the best source for the assistant |

**How we keep it cheap.** Most of the work is typed decisions with a confidence threshold, not free text. A known question gets the official text without a model; a coverage question goes to the model, with a citation; a request for a recommendation, or low confidence, goes to the contact center. A second typed check blocks answers that recommend. These decisions sit behind one interface, so the model can change without rebuilding. The index lives in PostgreSQL, which is already approved ([AS-14](../../ASSUMPTIONS.md)).

| Model | Cost to run per year | Decision |
|---|---|---|
| Small model of the tenant service | USD 3,000–14,000, database included | Default |
| Large model of the tenant service | Up to ≈ USD 96,000, model only | Only where the small one fails the spike thresholds |
| Open model on our own GPU machines | ≈ USD 17,500–70,000 for two machines, before operation | Rejected: costs more at this volume and needs a new approval |
| Specialized decision model (for example Jev) | Very low | Not now: external, outside the tenant, early access. Choosing bands with confidence is its use case, so pre-fill waits for one inside the tenant |

## 3. Spike: 15 Mar – 9 Apr 2027

One back-end developer and one MX product expert. Scope: current MX product documents in Spanish; at least 300 real coverage questions from the MX contact center, anonymized; the small model, and the large one only for comparison. No customers, no personal data, no pre-fill.

| Metric | Go | No-go |
|---|---|---|
| Correct and complete answers, graded by product experts | ≥ 90% | < 80% |
| Answers that cite the right section | ≥ 95% | < 90% |
| Recommendations shown, in ≥ 100 adversarial prompts reviewed by Compliance | 0 | ≥ 1 |
| Personal data in logs and telemetry | 0 | ≥ 1 |
| Time to the first answer, p95 | < 3 s | — |
| Model cost per conversation | ≤ USD 0.01 | > USD 0.08 (the cap at 100,000 conversations per month) |

Between the thresholds: one more two-week iteration, only once.

| Gate | Condition |
|---|---|
| 9 Apr: internal pilot | Every spike metric at "go" |
| 15 Jul: phase 1 for MX customers | In the pilot, agents accept ≥ 80% of the answers unchanged, with zero recommendations and zero personal data in logs; Security approves customer use; Compliance signs off. It starts with a share of customers, to measure avoided calls |
| Three months later: keep it, add CO and CL | Avoided calls × USD 4.10 ≥ the yearly cost to run; otherwise we turn it off |
| Phase 2: pre-fill | A specialized decision model inside the tenant, approved by Security, chooses ≥ 95% of the bands correctly in a test set |

## 4. Risks of a customer-facing LLM, and controls

| Risk | Control |
|---|---|
| Wrong answer about a coverage | Answers only from the approved documents of the country and product, with a citation; otherwise the contact center. AI notice and an option to talk to a person |
| Recommending a coverage | Typed check on every answer; adversarial set as a pipeline gate; Compliance review per country |
| Prompt injection | No tools that write data. Pre-fill proposes an answer band ([AS-01](../../ASSUMPTIONS.md)) only with high confidence; the customer confirms each field, and the price comes from rating |
| Personal data (C5) | Redaction before logs; no training with customer data; index and conversations inside the country data boundary |
| Old documents | Versioned index, rebuilt on publication; the answer shows the document version |
| Impact on the quote journey (C1, C2) | Separate component with its own bulkhead; the journey continues if it fails; a flag per country turns it off |
| Model change or cost increase | Pinned model version; evaluation suite before any change; limits per session; daily cost alert |

Rules for teams that build LLM features: [`docs/ai/AI_GUIDELINES.md` §5](../ai/AI_GUIDELINES.md#5-building-features-that-call-an-llm).
