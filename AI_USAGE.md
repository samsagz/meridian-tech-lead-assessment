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
| A | Challenge my approach to quoting while a Salesforce org is down | My decision: a price grid computed in Salesforce, live rating as fallback, and payment in the session (AS-01 to AS-03) | Counterarguments traced to `SALESFORCE_CONSTRAINTS.md`; retries checked against C2 | Its cap on combinations and its 06:00 grid window, because both contradicted the evidence |
| C | Options for severity, actions, SLOs, alerting and deployment; first drafts of the postmortem and `SLOS.md` | Severity, actions, SLO targets and rollout, decided by me after its challenges | Numbers recomputed from the CSV files and logs; alert thresholds replayed on the 30 Sep data; diagrams rendered | Its first severity call, a long timeline table, a single 99.5% target that current products could never meet, and its country-by-country rollout |

## Reflection

Written at the end of the assessment.
