# AI guidelines: Direct Digital Channel cell

For every person who writes code for the cell: the internal team and the Nimbus Digital developers (`TEAM.md`). They cover coding assistants, test generation, documentation and code review. The Technical Lead owns them and reviews them every quarter, first in March 2027 (§6).

The base rule comes from [`ENGINEERING_STANDARDS.md` §3](../engineering/ENGINEERING_STANDARDS.md#3-code-review-and-branching): AI assistance is declared, and the author answers for every line.

## 1. Allowed tools

| Tool | Status | Conditions |
|---|---|---|
| LLM service in Meridian's Azure tenant (`BUSINESS_REQUESTS.md:47`) | Allowed | Access with a Meridian identity, Nimbus included. An IDE plug-in is allowed only if it calls the tenant endpoint and Security lists it ([assumption AS-21](../../ASSUMPTIONS.md)) |
| Any other AI tool: personal or free accounts, vendor tools (for example, the assistant behind `PR-388.md:25`) | Not allowed with Meridian code or data | — |
| A coding assistant with an enterprise contract | Decided in the March 2027 review, with data from §6 | No retention and no training with our code, SSO, audit logs, approval of the Security office |

We start with the tenant service because it is already approved and keeps prompts inside Meridian (AS-21). One tool also gives one source of telemetry to measure §6.

## 2. Data

| May be shared | Never shared |
|---|---|
| Code and documentation of Meridian repositories | Personal data (C5), also inside the tenant, because one service serves developers of the three countries |
| OpenAPI contracts and ADRs | Production data and records of the Salesforce orgs |
| Synthetic test data | Secrets, tokens, connection strings and production configuration values (C5, [`REVIEW.md` #1](../review/REVIEW.md#1-findings)) |
| Logs after redaction | Open security findings, such as penetration test results |
| Public documentation | Third-party code under NDA, and code of other Nimbus customers |

When a developer is not sure, they ask the Technical Lead before sharing. A secret pasted in a prompt by mistake is treated as exposed and rotated (security gate, [`ENGINEERING_STANDARDS.md` §2](../engineering/ENGINEERING_STANDARDS.md#2-delivery-flow-and-quality-gates)).

## 3. Review and disclosure before merge

Every PR has a required field, "AI assistance": none, partial or major, and which parts (code, tests, documentation). Declaring has no penalty, because the declaration is what lets us measure. A false declaration is a review finding, like the checklist of PR-388 ([`REVIEW.md` #15](../review/REVIEW.md#1-findings)).

| Output | Required before merge |
|---|---|
| Any change | The same gates as any change ([`ENGINEERING_STANDARDS.md` §2](../engineering/ENGINEERING_STANDARDS.md#2-delivery-flow-and-quality-gates)). The author can explain every line in the review |
| Production code | The area of the change triggers the second approval, not the tool: contracts, identity, personal data, Salesforce calls and shared country configuration ([`ENGINEERING_STANDARDS.md` §3](../engineering/ENGINEERING_STANDARDS.md#3-code-review-and-branching)). Generated security code (authentication, authorization, cryptography, secret handling) always needs it |
| Tests | They must be able to fail: they do not mock the class under test ([`REVIEW.md` §4](../review/REVIEW.md#4-feedback-to-the-author), point 7), and the mutation gate applies ([`ENGINEERING_STANDARDS.md` §2](../engineering/ENGINEERING_STANDARDS.md#2-delivery-flow-and-quality-gates)). The reviewer checks that the assertions verify behavior, not only coverage |
| New dependencies suggested by AI | The author checks that the package exists, is maintained and has the expected name. The license gate of §4 applies |
| Documentation, ADRs and runbooks | The author checks every fact and number against its source |
| AI-assisted review | Allowed as extra comments. It never counts as an approval |

## 4. Accountability, security and intellectual property

| Topic | Rule |
|---|---|
| Accountability | The author answers for every line; the reviewer, for what they approve; the Technical Lead, for these guidelines. The Nimbus delivery lead answers for the vendor developers, checked in the quarterly business review ([`ENGINEERING_STANDARDS.md` §4](../engineering/ENGINEERING_STANDARDS.md#4-nimbus-digital-c8)) |
| Security | Generated code passes the same SAST, secret-scanning and SCA gates ([`ENGINEERING_STANDARDS.md` §2](../engineering/ENGINEERING_STANDARDS.md#2-delivery-flow-and-quality-gates)). Prompts never include secrets (§2) |
| Ownership | Meridian owns all code written for it, generated code included. This is a clause of the Nimbus renewal (C8, [`ENGINEERING_STANDARDS.md` §4](../engineering/ENGINEERING_STANDARDS.md#4-nimbus-digital-c8)) |
| Licenses | The SCA gate checks licenses against an allow-list. GPL and AGPL block in deployed components; other copyleft licenses need Legal. Frontends count, because their JavaScript reaches the browser |
| Copied code | If generated code looks like a known library, we use the library. The protected-material filter of the tenant service is on, if the service offers it (AS-21). A snippet scan runs only when code leaves Meridian: the Nimbus exit plan or code we publish |

## 5. Building features that call an LLM

The risks and controls of a customer-facing assistant are in [`FEASIBILITY_GENAI.md` §4](../roadmap/FEASIBILITY_GENAI.md#4-risks-of-a-customer-facing-llm-and-controls). Any LLM feature of the cell also follows these rules:

| Control | Rule |
|---|---|
| Prompts and model | Prompts live in the repository and are reviewed as code; the model version is pinned |
| Evaluation suite | A pipeline gate for every change of prompt, model or documents, with the spike thresholds ([`FEASIBILITY_GENAI.md` §3](../roadmap/FEASIBILITY_GENAI.md#3-spike-15-mar--9-apr-2027)) |
| Model output | Untrusted input: validated against a schema; no tools that write data; it never sets a price |
| Personal data | Redacted before logs and telemetry; it stays inside the country data boundary (C5) |
| Operation | Own bulkhead and a flag per country; cost limit per session and a daily cost alert; Security approval before any customer use (`BUSINESS_REQUESTS.md:47`) |

## 6. Rollout, enforcement and measurement

| When | Step |
|---|---|
| Dec 2026 | Review of this draft with the Security office and Legal; AS-21 confirmed |
| Jan 2027 | One-hour session for the cell and Nimbus, using PR-388 as the example; PR template with the AI field; Meridian identities for the Nimbus developers |
| 31 Jan 2027 | Nimbus renewal: tools, data and code ownership as contract terms (C8) |
| Mar 2027 | First review with data. If adoption is low or there are signs of other tools, we propose an enterprise coding assistant to Security |

| Enforcement | How |
|---|---|
| AI field in every PR | Required PR check in the branch policy: the merge is blocked if the field is empty |
| Monthly sample | The Technical Lead reviews 10 merged PRs: the declaration matches the tool telemetry of the author, and the tests can fail |
| Nimbus | Contract clause and quarterly business review |

| Metric | Target | Source |
|---|---|---|
| PRs with the AI field filled | 100% | PR check |
| Secrets or personal data sent to AI tools | 0 | Tenant service audit log, secret scanning |
| Change failure rate of PRs with AI assistance | Not higher than the other PRs | Pipeline and the AI field |
| Mutation score of generated tests | ≥ 60%, as for all code | Pipeline |
| Tests that mock the class under test | 0 in the sample | Monthly sample |
| Adoption and use of other tools | No target: an input for the March review | Tenant telemetry, quarterly survey, declarations |
