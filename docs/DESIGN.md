# Metro Real Estate Intelligence Platform: Design Doc
Author: <your name> · Date: 2026-10-15 · Status: Draft

## 1. Context
<!-- What does a real estate analyst do today to compare rents, rates, and
     construction across Chicago, the DMV, and Dallas? What's slow, manual,
     or scattered? Draw on your internship. -->
     Real estate investment teams spend hours stitching together fragmented housing, economic, and permit data across dozens of jurisdictions, and still learn about competing supply too late, since the earliest signs are buried in city council and zoning documents nobody has time to read. Analysts carry most of the burden, while partners and investment committees wait on them for answers and get numbers they can't really trace or trust. Smaller firms feel it most, because they have the same questions as large firms without the budget for enterprise data or research staff.

## 2. Goals
<!-- Make each one checkable. Examples of the shape (write your own):
     "Load X sources for 3 metros on a schedule with no manual steps."
     "Adding a metro requires a config change only." -->

## 3. Non-goals
<!-- What are you deliberately not building? Consider: real-time data,
     individual property listings, price predictions, user accounts,
     a public product. Why leave each one out? -->

## 4. Architecture overview
<!-- Add the diagram here (export it from your build guide, or redraw it).
     Then one paragraph, left to right: where data starts, where it lands
     raw, how it's transformed, where AI and agents act on it, and who
     consumes the result. -->

## 5. Data sources
| Source | What it provides | Frequency | Method |
|---|---|---|---|
| FRED | | | |
| Zillow Research | | | |
| Census ACS | | | |
| HUD FMR | | | |
| City permit portals | | | |
<!-- How often does each source actually update? Pull on that schedule,
     not more. Which are APIs and which are file downloads? -->

## 6. Key decisions
<!-- One or two lines each, linking to the ADR in DECISIONS.md.
     Candidates: DuckDB locally before cloud; dbt for transformations;
     batch over streaming; starting with one metro; AWS vs. GCP. -->

## 7. Data model
<!-- Which fact tables and dimensions? For each fact, finish the sentence
     "one row = one ___." How will weekly rates and monthly rents meet? -->

## 8. Reliability and data quality
<!-- How do you make sure nothing is lost or duplicated?
     - Idempotency: what happens if a run executes twice?
     - Dead-letter handling: where do records go that fail parsing or
       validation, so they're kept for review instead of dropped?
     - Reconciliation: how do you confirm row counts in the warehouse
       match what the source sent?
     - Tests: which dbt tests block a run, and which only warn? -->

## 9. Security and access
<!-- Where do API keys live, and how do you keep them out of Git?
     What roles exist (pipeline, read-only analyst, AI agents), and what
     is each allowed to read or write? Why do agents get the least access? -->

## 10. Cost
<!-- Rough monthly estimate for storage, warehouse compute, and LLM calls.
     What caps it? Think billing alarms, caching AI results, step and
     budget limits on agents, auto-suspending the warehouse. -->

## 11. Risks and open questions
<!-- Examples to consider: messy and inconsistent permit data across cities,
     Zillow vs. Census metro IDs not matching, source format changes,
     data terms of use, your own time. What haven't you decided yet? -->

## 12. Milestones
<!-- One line per phase from your build guide, written as an outcome:
     "Phase 1: Chicago rate and rent data lands raw, safely, on demand." -->