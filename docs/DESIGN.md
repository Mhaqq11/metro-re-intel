# Metro Real Estate Intelligence Platform: Design Doc
Author: <Menelik Haqq> · Date: 2026-10-15 · Status: Draft

## 1. Context
<!-- What does a real estate analyst do today to compare rents, rates, and
     construction across Chicago, the DMV, and Dallas? What's slow, manual,
     or scattered? Draw on your internship. -->
     Real estate investment teams spend hours stitching together fragmented housing, economic, 
     and permit data across dozens of jurisdictions, and still learn about competing supply too late, 
     since the earliest signs are buried in city council and zoning documents nobody has time to read. 
     Analysts carry most of the burden, while partners and investment committees wait on them for answers 
     and get numbers they can't really trace or trust. Smaller firms feel it most, because they have the 
     same questions as large firms without the budget for enterprise data or research staff.

## 2. Goals
<!-- Make each one checkable. Examples of the shape (write your own):
     "Load X sources for 3 metros on a schedule with no manual steps."
     "Adding a metro requires a config change only." -->
     - Ingest 5+ public sources for 3 metros on a schedule
     - Rerunning any pipeline run creates zero duplicate rows.
     - Every mart table is covered by dbt tests; failing critical tests block publishing 
     - Adding a new metro requires only a config change (metro.yaml), no code changes
     - Classify permit descriptions with AI at a measured accuracy above a rules-based baseline
     - Agents diagnose planted pipelines failures correctly in at least 80% of cases.
     - Keep cloud and AI costs under $15/month

## 3. Non-goals
<!-- What are you deliberately not building? Consider: real-time data,
     individual property listings, price predictions, user accounts,
     a public product. Why leave each one out? -->
     - Real-time data. Sources update weekly to yearly; batch is enough
     - Individual property listing. Scraping listing sites violoates their terms.
     - Price predictions or investment advice. The platform describes markets; it doesn't recommend buys.
     - User accounts or a public product. This is a portfolio system for a small set of users.
     - Fully autonomous agents. Agents investigate and propose; humans approve any change

## 4. Architecture overview
<!-- Add the diagram here (export it from your build guide, or redraw it).
     Then one paragraph, left to right: where data starts, where it lands
     raw, how it's transformed, where AI and agents act on it, and who
     consumes the result. -->



## 5. Data sources
<!-- How often does each source actually update? Pull on that schedule,
     not more. Which are APIs and which are file downloads? -->
     -----------------------------------------------------------------------------------------------------------------------------------------
     |    Source          |                 What it provides                             |    Frequency |           Method                   |
     -----------------------------------------------------------------------------------------------------------------------------------------
     |FRED                |         30-year mortgage rate, inflation                     |    Weekly    |    REST API (JSON, free key)       |
     -----------------------------------------------------------------------------------------------------------------------------------------
     |Zillow Research     |         Rent Index(ZORI), home values(ZHVI)                  |    Monthly   |    Bulk CSV download               |
     -----------------------------------------------------------------------------------------------------------------------------------------
     |Census ACS          |         Income, population, demographics                     |    Annual    |    REST API(free key)              |
     -----------------------------------------------------------------------------------------------------------------------------------------
     |HUD Fair Market     |         Government rent benchmarks by bedroom count          |    Annual    |    Bulk file download              |
     -----------------------------------------------------------------------------------------------------------------------------------------
     |City permit portals |         Building permits(Chicago, Dallas, DC)                |    Daily     |    Open data APIs (Socrata, ArcGIS)|
     -----------------------------------------------------------------------------------------------------------------------------------------


## 6. Key decisions
<!-- One or two lines each, linking to the ADR in DECISIONS.md.
     Candidates: DuckDB locally before cloud; dbt for transformations;
     batch over streaming; starting with one metro; AWS vs. GCP. -->
     - DuckDB locally before a cloud warehouse: fast, free iterations; dbt makes the switch cheap
     - dbt for transformations: tested, documented, layered SQL instead of ad hoc scripts
     - Batch over streaming: matches how often sources update, at a fraction of the complexity
     - Start with one metro, exand through config: prove the pipeline before scaling it
     - Raw data stored untouched; any transformation bug can be fixed by reprocessing


## 7. Data model
<!-- Which fact tables and dimensions? For each fact, finish the sentence
     "one row = one ___." How will weekly rates and monthly rents meet? -->

     Star schema with conformed dimensions.
     -------------------------------------------------------------------------------------------------------------------------
     |Table                         |    Type      |        Grain (one row = …)                                              |
     -------------------------------------------------------------------------------------------------------------------------
     |fact_rent_monthly             |    Fact      |   one geography, one month                                              |
     -------------------------------------------------------------------------------------------------------------------------
     |fact_rates_weekly             |    Fact      |	one rate series, one week                                             |
     -------------------------------------------------------------------------------------------------------------------------
     |fact_permits                  |    Fact      |   one building permit                                                   |
     -------------------------------------------------------------------------------------------------------------------------
     |fact_demographics_annual      |    Fact      |   one geography, one survey year                                        |
     -------------------------------------------------------------------------------------------------------------------------
     |fact_fmr_annual               |    Fact      |   one geography, one fiscal year, one bedroom count                     |
     -------------------------------------------------------------------------------------------------------------------------
     |dim_geography                 |    Dimension |	one metro, county, or ZIP, with a crosswalk of source IDs             |
     -------------------------------------------------------------------------------------------------------------------------
     |dim_date                      |    Dimension |   one calendar day                                                      |
     -------------------------------------------------------------------------------------------------------------------------
     |dim_permit_category           |    Dimension |   one AI-assigned permit category                                       |
     -------------------------------------------------------------------------------------------------------------------------
     |mart_metro_market_monthly     |    Mart      |   one metro, one month                                                  |
     -------------------------------------------------------------------------------------------------------------------------
## 8. Reliability and data quality
<!-- How do you make sure nothing is lost or duplicated?
     - Idempotency: what happens if a run executes twice?
     - Dead-letter handling: where do records go that fail parsing or
       validation, so they're kept for review instead of dropped?
     - Reconciliation: how do you confirm row counts in the warehouse
       match what the source sent?
     - Tests: which dbt tests block a run, and which only warn? -->
     - Idempotency: raw files are written to deterministic, date-partitioned paths, and 
     warehouse loads merge on natural keys, so reruns overwrite rather than duplicate 

     - Revisions: each run re-pulls a lookback window to catch corrected historical values

     - Dead-letter handling: records that fail parsing or validation go to a quarantine table 
     with the reason and source file, never silently dropped 

     - Reconciliation: each run compares rows recieved from the source with rows loaded, and alerts on mismatch
     
     - Tests: Uniqueness, not null, relationships, and freshness on every model; key and freshness failures block,
     others warn

     - Failures: tasks retry with backoff, then alert and trigger the reliablility agent


## 9. Security and access
<!-- Where do API keys live, and how do you keep them out of Git?
     What roles exist (pipeline, read-only analyst, AI agents), and what
     is each allowed to read or write? Why do agents get the least access? -->
     - Secrets:.env locally. No keys in code or GIT.
     - Roles (least privilege):
          - pipeline: writes raw and warehouse schemas
          - analyst: reads marts only
          - agent: reads marts and logs: can open Github issues and draft pull requests;
          cannot merge, delete, or write data.
     - The data assistant run SELECT-only queries on approved marts, with row limitation and timeouts
     - Permit text and error messages are treated as untrusted input to agents

## 10. Cost
<!-- Rough monthly estimate for storage, warehouse compute, and LLM calls.
     What caps it? Think billing alarms, caching AI results, step and
     budget limits on agents, auto-suspending the warehouse. -->
     - Target under $15/month, with billing alarms at $5 and $15
     - Storage: raw data is small; S3 cost is minimal
     - Warehouse: Development on DuckDB(free); cloud warehouse auto suspends when idle
     - AI: permit classifications are cached by text hash, so each description is proccessed once;
     requests are batched and cost is logged per run.
     - Agents: hard caps on steps,tokens, and dollars per run.

## 11. Risks and open questions
<!-- Examples to consider: messy and inconsistent permit data across cities,
     Zillow vs. Census metro IDs not matching, source format changes,
     data terms of use, your own time. What haven't you decided yet? -->
     - Zillow, Census, and HUD use different geographic IDs; a bad crosswalk silently corrupts joins
     - Permit data differs by city in columns, categories, and quality
     - LLM accuracy may not beat the rules baseline enough to justify cost
     - Agents could be misled by malicious text in inputs (prompt injection)

## 12. Milestones
<!-- One line per phase from your build guide, written as an outcome:
     "Phase 1: Chicago rate and rent data lands raw, safely, on demand." -->
     - Phase 0 (Week 1): Repo, Accounts, and project structure set up
     - Phase 1 (Weeks 2-3): Chicago rate and rent data lands raw, safely, with one command.
     - Phase 2 (Weeks 4-6): Tested dimensional model and first mart in DuckDB
     - Phase 3 (Weeks 7-8): Scheduled, cloud-deployed pipeline covering all three metroes and sources.
     - Phase 4 (Weeks 9-10): AI permit classification with measured accuracy
     - Phase 5 (Weeks 11-12): Reliability and market brief agents with guardrails and evaluation.
     - Phase 6 (Weeks 13-14): Dashboard, ask-your-data assistant, and public write-up
