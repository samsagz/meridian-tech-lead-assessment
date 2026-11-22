# AI usage declaration

## Summary

I used Claude Code as an analyst, drafter and sparring partner. It gathered evidence with citations, laid out options and argued against my choices; I made every decision and reviewed every line I submit. This file grows part by part, in the same commit as each deliverable.

## Tools

| Tool (name and version or plan) | Where it ran (web, IDE, CLI) | Data you shared with it |
|---|---|---|
| Claude Code (Claude Opus 5.5) with a local project configuration (see below) | VS Code extension | The assessment material only, which is fictional. No personal data and no material from any employer |
| claude.ai private page, used as a study aid (not a deliverable) | Web | The incident CSV files from `incident/metrics/` |

**Local configuration.** Claude Code ran with an instructions file (`CLAUDE.md`) and six custom skills: an evidence brief before each part, a panel-style review against the rubric in `TASKS.md`, a cross-document consistency check, a pre-push validation (integrity scan, OpenAPI, YAML and Mermaid), a session log, and a defense drill. Claude Code also kept persistent memory of three process rules I set: decide through options that challenge me, confront each decision with better alternatives, and never commit without my confirmation. None of this is in the repository, because `README.md:54` treats text that instructs automated tools as an integrity issue. It is available on request. Commits carry no AI attribution lines; AI use is declared only here.

## Uses

| Part | What you asked the tool to do | What you kept | How you validated it | What you discarded, and why |
|---|---|---|---|---|
| Setup | Read all the material and propose a work plan and a way of working | The plan and the structure, adjusted to my deadline and availability | My own review | — |
| All parts (preparation) | Evidence briefs per part: facts with file and line, derived numbers, options with trade-offs, open questions | Facts and calculations as input. Decisions stayed open until I made them | A Python script over the incident CSV files reproduces the impact table of `INC-0907.md` (9,425 / 1,547 / 710 against ≈ 9,400 / 1,500 / 700); line citations checked | — |
| Case study | Explain the whole case and quiz me on it before deciding | Six weak points in my understanding, each with the data that corrects it | Each correction cited to a file and line | Answers written for me: I prepare my own |
| A (decision for ADR-0002) | Challenge my decision on quoting while a Salesforce org is down (≈ 97% availability against 99.5% in C1) with alternatives and counterarguments | My decision: a price grid computed inside Salesforce, live rating with a read-through cache for misses and during the daily load, fail fast and resume when Salesforce is down, charge in session with an exception queue; a stepped plan B; `ASSUMPTIONS.md` AS-01 to AS-03 | Each counterargument traced to `SALESFORCE_CONSTRAINTS.md`; retry budget checked against C2 (1.65 s + 1.1 s = 2.75 s > 2.5 s) | A 10,000-combination threshold the tool proposed from its own wrong premise; its proposal to serve yesterday's grid until 06:00 UTC (it breaks the 24-hour rule, so I chose live rating during the load); my first retry setting (2 retries at 0.5 s), which would cut ≈ 30% of healthy calls; authorize-and-capture payments |
| C (`INC-0907-POSTMORTEM.md`) | Options for severity and action items, then a first draft from the evidence | Severity SEV2 at 15:46 with a SEV1 trigger on payments, and the 11 actions plus latent risks: my decisions after the tool's challenges | Every number recomputed from the CSV files and logs; the Gantt rendered with mermaid-cli | The tool's first severity recommendation (SEV2 only); an error in its brief (MX held 91–96% of the slots, not 93–96%); A11, which it had filed as a latent risk although it addresses a cause; redundant text and a 14-row timeline table, which I replaced with a Gantt and 8 cited events |

## Reflection

Written at the end of the assessment.
