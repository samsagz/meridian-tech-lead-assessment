# AI usage declaration

## Summary

I used Claude Code as analyst, drafter and sparring partner. For each part it prepared an evidence brief with citations and options. I made every decision; on the main ones, it first argued against my choice. It drafted the documents from my decisions, and I revised every line. The table lists only what is specific to each part.

## Tools

| Tool (name and version or plan) | Where it ran (web, IDE, CLI) | Data you shared with it |
|---|---|---|
| Claude Code (Claude Opus 5.5) with a local project configuration (see below) | VS Code extension | The assessment material only, which is fictional. No personal data and no employer material |
| claude.ai private page, used as a study aid (not a deliverable) | Web | The incident CSV files from `incident/metrics/` |

**Local configuration.** A project instructions file (`CLAUDE.md`), six custom skills (evidence brief, panel-style review, consistency check, pre-push validation, session log, defense drill) and persistent memory of my process rules (for example, no commit without my confirmation). They are not in the repository because `README.md:54` treats text that instructs automated tools as an integrity issue; they are available on request. Commits carry no AI attribution; AI use is declared only here.

## Uses

| Part | What you asked the tool to do | What you kept | How you validated it | What you discarded, and why |
|---|---|---|---|---|
| Preparation | Read the material, plan the work, brief each part, quiz me on the case | Briefs as input; my weak points to study | A script reproduces the impact table of `INC-0907.md`; citations checked | — |
| A | Challenge my approach to quoting while a Salesforce org is down; options for the reuse model, the MX route and issuance; first drafts of both ADRs and `ARCHITECTURE.md` | My decisions: a price grid computed in Salesforce, live rating as fallback and payment in the session (AS-01 to AS-03); one regional deployment with bulkheads and data per country (AS-06); issuance in Azure, because MuleSoft is the bottleneck | Counterarguments traced to `SALESFORCE_CONSTRAINTS.md`; retries and timeouts checked against C2; usage and peak figures recomputed from the CSV files; diagrams rendered | Its cap on combinations and its 06:00 grid window, because both contradicted the evidence; its deployment per country, because I extended the pattern that CO and CL already run; a separate cache for live prices, because nothing per quote should stay in memory; PostgreSQL as the first store for the grid, because I check the MuleSoft Object Store first (AS-07) |
| B | Options to roll out by country with one regional deployment; first drafts of the standards and the pipeline template | My decisions: deploy once and release each country with flags, approved by the Product Owner; thresholds taken from `SLOS.md`; Nimbus terms tied to the renewal (AS-15 to AS-19) | Template parsed as YAML; each gate traced to `DELIVERY_METRICS.md`, PR-388 or INC-0907; thresholds checked against `SLOS.md` and the ADRs | A traffic shift routed by country, because it reopened the rollout I chose in Part C; the brief's first thresholds, because they did not match my SLOs |
| C | Options for severity, actions, SLOs, alerting and deployment; first drafts of the postmortem and `SLOS.md` | Severity, actions, SLO targets and rollout, decided by me after its challenges | Numbers recomputed from the CSV files and logs; alert thresholds replayed on the 30 Sep data; diagrams rendered | Its first severity call, a long timeline table, a single 99.5% target that current products could never meet, and its country-by-country rollout |
| D | Read PR-388 against the codebase and list the issues with their lines; first drafts of the review and of the feedback | The verdict, contract first with a stub to unblock the vendor, and no mention of the incident in the feedback | Line numbers rebuilt from the diff with a script; each finding checked against the codebase; feedback counted (389 words) | Its 19-row table, merged to 15 so the author sees the main problems first |
| E | Score the backlog, check it against the capacity per pool, give options for the Q1 gap, the unsupported MX stack, the launch dates and BR-02; first drafts of the roadmap, the feasibility analysis and the memo | My decisions: simplified WSJF, Nimbus on Home Essentials from February, an upgrade of the MX back end only, go-live before the month-end freeze; for BR-02, a test in two phases with a small model, typed decisions with confidence thresholds, and a yearly cost cap (AS-08 to AS-14) | Capacity and demand recomputed from `TEAM.md` and the team estimates; call costs and break-even recomputed from `BUSINESS_REQUESTS.md`; Gantt rendered | Its first estimate of a single customer build of 12–18 pw, because I asked for cheaper paths; a self-hosted model, because it costs more at this volume; plain-language summaries as a replacement for the assistant, which I keep only as a complement |

## Reflection

Written at the end of the assessment.
