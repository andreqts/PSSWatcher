# Opportunity Watch Specification

## Problem Statement

Academic hiring opportunities (temporary lecturer, effective/assistant professor, post-doc) in electrical/electronic/energy/computer engineering, software engineering, and computer science, in Foz do Iguaçu (PR, Brazil), are scattered across several university sites with no central feed. Checking each site manually every day is tedious and easy to miss. This feature automates that check and publishes a single, public, low-noise report.

## Goals

- [ ] Every day at 08:00 BRT, automatically discover currently-open matching opportunities from a configured list of site searches.
- [ ] Publish the current set of open opportunities as a public report with zero manual steps.
- [ ] Notify a fixed list of interested people by email only about opportunities that are new since the last run (no repeat noise).
- [ ] Keep false positives low via a Jev (OpenRouter's TypeSafe decision model) classification pass before anything is published or notified.
- [ ] Run entirely on free-tier infrastructure (GitHub Actions free minutes, free DuckDuckGo search, a free Gmail account, low-cost OpenRouter credits).
- [ ] Alert the maintainer by email whenever a run had any failure or exclusion, so a real opportunity is never silently lost to an overlooked error.

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
| --- | --- |
| Self-serve subscription (public sign-up form/button) | User explicitly deferred to backlog for a future iteration; recipients are registered manually for now. Candidate future design: Cloudflare Worker + Turnstile captcha writing to the recipient store. |
| GitHub Pages / dedicated static site for the report | Deferred; README.md rendering on the repo page is the v1 report surface (YAGNI - can flip on Pages later without changing the data model). |
| Paid search API (Google Custom Search, SerpAPI) | DuckDuckGo (free, no key) is the chosen search backend for v1. |
| External database for dedup state | `data/seen.json` committed to the repo is sufficient at this volume and free. |
| Multi-language / multi-region expansion (other cities/states, other fields) | Out of scope; the query list is fixed to the areas and city named by the user, but is a config file so it's extendable later. |

---

## Assumptions & Open Questions

Every ambiguity is resolved or recorded here - nothing is left silently unclear.

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Search backend | DuckDuckGo via the `ddgs` Python package, no API key | Free, no key, supports `site:` queries; Google scraping gets blocked/CAPTCHA'd (Q5) | y |
| Dedup / state persistence | `data/seen.json`, committed back to the repo by the workflow each run | Free, simple, versioned automatically, no external DB (Q6) | y |
| Report surface | Rewrite a marked section of `README.md` in place | Renders automatically on GitHub, zero setup; GitHub Pages is a later upgrade if wanted (Q7) | y |
| Email delivery | Gmail SMTP (`smtplib`, stdlib) via a dedicated free Gmail account + App Password | Free, no new dependency, keeps the App Password scoped to this project (Q8) | y |
| Recipient list storage | A single GitHub Actions secret (newline-separated emails), edited manually in repo Settings | Repo is public; a committed file would leak emails into git history forever (Q9) | y |
| Self-serve subscription | Deferred to backlog; manual registration only in v1 | User explicitly chose to defer rather than add a serverless backend now (Q11) | y |
| Repo visibility | Public | User's stated intent - the report must be publicly visible (Q9) | y |
| Validation method | OpenRouter's Jev decision model (`typesafe/jev-latest`), a purpose-built fast/cheap classifier - not a general chat-completion LLM prompt | User explicitly wants Jev, not an LLM; verified live against OpenRouter's own docs on 2026-09-22 (openrouter.ai/typesafe, openrouter.ai/docs/cookbook/evaluate-and-optimize/jev-verified-cascade) | y |
| Validation fallback | If OpenRouter's Decisions API (the alpha-status endpoint Jev is served through) proves unreliable at implementation time, retry the classification via OpenRouter's `openrouter/free` Free Models Router (auto-selects among free-tier models only) - never `openrouter/auto` and never a paid model. If the free-tier retry also fails, the candidate is excluded and the failure logged (same handling as any other validation failure) | Jev's serving endpoint is explicitly labeled alpha by OpenRouter; user wants zero spend risk from the fallback path - only Jev itself may use paid credits. `openrouter/auto` was considered but OpenRouter's own docs confirm it has no free-only restriction and can route to (and bill) paid models, which would defeat that guarantee | y |
| Exact model/pricing | Between `typesafe/jev-1.13` and `typesafe/jev-latest`, picked against current OpenRouter pricing at Design time | Not fabricated now; user has pre-loaded paid credits and prioritized reliability over $0 cost (Q10) | y |
| Schedule | Daily cron at `11:00 UTC` (08:00 America/Sao_Paulo, fixed UTC-3, no DST since 2019) | User-specified trigger time (Q1/Q2, corrected from an initial PM/AM typo) | y |
| Opportunity identity (dedup key) | SHA-256 hash of the normalized result URL | No natural ID exists in scraped results; URL is the only stable identifier across runs | n (agent default, flag if wrong) |
| Validation-failure handling | If the OpenRouter call errors for a candidate, exclude it from this run's report/notification rather than include it unvalidated | Under-reporting one run is safer than publishing an unverified false positive to a public page | n (agent default, flag if wrong) |
| Stale-opportunity removal | An opportunity is dropped from the README report after 3 consecutive runs where it no longer appears in search results, but its dedup ID is kept in `seen.json` for 30 days to avoid re-notifying on search flakiness | Not explicitly discussed; a reasonable default balancing "report reflects reality" against "don't spam on a transient search miss" | n (agent default, flag if wrong) |
| Concurrent runs | Prevented via a GitHub Actions `concurrency` group on the workflow | Manual re-triggers could otherwise race with the scheduled run and corrupt `seen.json`/README | y (technical safeguard, not a product decision) |
| Maintainer failure alerts | A dedicated GitHub Actions secret (separate from the opportunity-recipient secret) holds a maintainer alert list - one or more newline-separated addresses, same format as the recipient list; one email per run to every address on it, only when that run had a recorded failure/exclusion | User is worried about silently losing an opportunity to an overlooked soft failure, and wants to be able to register more than one maintainer address. GitHub Actions already emails repo admins/watchers for free on a *hard* workflow failure (uncaught crash, non-zero exit) - that's an existing safety net requiring no work here. This feature closes the other gap: a run that succeeds overall but silently excluded a candidate (e.g. Jev + free-fallback both failed for one entry) | y |

**Open questions:** none - all resolved or logged above. Three rows are marked "agent default, flag if wrong" - call these out explicitly if any should be revisited before implementation.

---

## User Stories

### P1: Daily discover, validate, publish, notify ⭐ MVP

**User Story**: As an academic job seeker, I want the system to automatically search the configured university sites every day, filter out false positives, publish the current open opportunities publicly, and email me only about the new ones, so I never have to check multiple sites manually.

**Why P1**: This is the entire value of the product - without it there is nothing to ship.

**Acceptance Criteria**:

1. WHEN the scheduled GitHub Actions workflow triggers at 11:00 UTC daily THEN the system SHALL run every configured site search query.
2. WHEN a search query returns results THEN the system SHALL extract, for each result, at minimum a title, a URL, and the source site.
3. IF a search result is missing a title or a URL THEN the system SHALL discard that result and continue processing the rest.
4. WHEN a candidate opportunity is extracted THEN the system SHALL classify it via OpenRouter's Jev decision model before it can appear in the report or a notification.
5. IF Jev's verdict classifies a candidate as a false positive THEN the system SHALL exclude it from the report and from any notification.
6. IF the Jev/OpenRouter call for a candidate fails (error, timeout, alpha-endpoint outage, or invalid response) THEN the system SHALL retry the classification using an OpenRouter model request restricted to free-tier pricing only - the fallback SHALL never invoke a paid model.
7. IF the free-tier fallback classification also fails, times out, or returns no usable verdict THEN the system SHALL exclude that candidate from the current run's report and notification, and log the failure.
8. The system SHALL compute a stable identifier for each validated opportunity as the SHA-256 hash of its normalized URL.
9. WHEN a validated opportunity's identifier is not present in the previous run's `data/seen.json` THEN the system SHALL classify it as new.
10. WHEN one or more opportunities are classified as new THEN the system SHALL send exactly one email listing only those new opportunities to every address in the configured recipient list.
11. IF no opportunities are classified as new in a run THEN the system SHALL send no email.
12. The system SHALL rewrite the marked opportunities section of `README.md` on every run to reflect the current full set of open (non-stale) opportunities, regardless of whether any are new.
13. WHEN a previously-tracked opportunity is absent from search results for 3 consecutive runs THEN the system SHALL remove it from the `README.md` report.
14. The system SHALL retain an opportunity's identifier in `data/seen.json` for 30 days after it was last seen before purging it, so a transient search miss does not trigger a duplicate "new" notification.
15. The system SHALL commit the updated `data/seen.json` and `README.md` back to the repository at the end of every run that changed either file.
16. IF the email send fails THEN the system SHALL log the failure and still complete the report commit (a notification failure never blocks publishing).
17. The system SHALL prevent overlapping runs via a GitHub Actions concurrency group on the workflow.
18. WHEN a run completes THEN the system SHALL log a summary (queries run, candidates found, validated count, new count, errors encountered) to the workflow run log.

**Independent Test**: Trigger the workflow manually (`workflow_dispatch`) against a small fixed query list; confirm README updates, `seen.json` updates and commits, and an email arrives only when a genuinely new entry is injected into the fixture data.

---

### P2: Configurable query list and recipient management

**User Story**: As the maintainer, I want the site search queries and the recipient list to live in config rather than code, so I can add a university site or a recipient without touching the script.

**Why P2**: Not required for the very first working run, but required before this is comfortably maintainable - and cheap to build alongside P1.

**Acceptance Criteria**:

1. The system SHALL read its list of site search queries from a version-controlled config file (e.g. `config/queries.yaml`), not from hardcoded strings.
2. The system SHALL read its recipient list from a GitHub Actions secret at runtime, never from a file committed to the repository.
3. WHEN a maintainer adds a query to the config file THEN the next scheduled run SHALL include it without any code change.

**Independent Test**: Add a new `site:` query to the config file, push, trigger a manual run, confirm the new site is queried (visible in the run log summary).

---

### P2: Failure-report email alerting

**User Story**: As the maintainer, I want a daily email whenever a run had any failure or exclusion, so I can check the logs before a real opportunity is silently lost - a hard workflow crash already emails me via GitHub Actions' own notifications, but a run that succeeds overall while quietly excluding one candidate would not otherwise reach me.

**Why P2**: Not required to demo the core loop, but directly addresses the maintainer's stated fear of overlooked failures, and reuses the email infrastructure already built for P1 - cheap to add.

**Acceptance Criteria**:

1. WHEN a run completes with one or more recorded failures (a failed search query, a candidate excluded because both Jev and its free-tier fallback failed, or a failed notification/report-commit step) THEN the system SHALL send one failure-report email to every address in the configured maintainer alert list.
2. IF a run completes with zero recorded failures THEN the system SHALL send no failure-report email.
3. The failure-report email SHALL list, for each recorded failure, its type, the affected query or candidate URL, and the error message.
4. The system SHALL read the maintainer alert list - one or more newline-separated addresses - from a GitHub Actions secret dedicated to this purpose, distinct from the interested-recipient secret (P1-AC10).
5. WHEN both a new-opportunity notification and a failure report are due in the same run THEN the system SHALL send them as two separate emails.
6. WHEN an unhandled exception occurs anywhere in the pipeline after required secrets (OpenRouter, Gmail) have loaded THEN the system SHALL make one best-effort attempt to send the failure-report email, naming the unexpected error, to the maintainer alert list before the process exits with a non-zero status.
7. IF an unhandled exception occurs before required secrets have loaded THEN the system SHALL make no attempt to send a failure-report email (no credentials exist yet); GitHub Actions' own built-in failure notification to repo admins/watchers is the sole safety net for this case.

**Independent Test**: Register two addresses in the maintainer alert list; force one candidate's Jev call and its free-tier fallback to both fail in a manual run; confirm exactly one failure-report email arrives at each of the two addresses naming that candidate, and confirm a clean run sends none. Separately, inject an unhandled exception into the state/dedupe step of a manual run (after secrets load) and confirm a failure-report email still arrives naming that error, and that the job still exits non-zero.

---

### P3: Run observability

**User Story**: As the maintainer, I want enough logging in each run to diagnose a bad day (search backend down, LLM erroring, email failing) without re-running locally.

**Why P3**: Nice-to-have robustness; the system is useful without it, but painful to debug without it.

**Acceptance Criteria**:

1. WHEN any external call (search, LLM validation, email send, git commit) fails THEN the system SHALL log the failing step, the target, and the error message to the workflow run log.
2. The system SHALL log the final per-run summary described in P1-AC17 even when one or more steps failed.

**Independent Test**: Force a failure (e.g. invalid OpenRouter key in a test run) and confirm the run log clearly names the failing step and still completes with a summary line.

---

## Edge Cases

- IF a search query returns zero results THEN the system SHALL treat it as zero candidates and continue with the remaining queries (not an error).
- IF the same opportunity URL is returned by more than one site query THEN the system SHALL deduplicate it to a single entry before validation (same hash).
- IF `data/seen.json` does not exist yet (first-ever run) THEN the system SHALL treat every validated opportunity as new.
- IF the DuckDuckGo backend errors or rate-limits for a query THEN the system SHALL log it, skip that query, and continue with the remaining queries rather than failing the whole run.
- IF the recipient-list secret is empty or unset THEN the system SHALL still update the README report but SHALL skip sending any email (log a warning, not an error).
- IF the script crashes before required secrets have loaded THEN no self-generated failure-report email is possible (no credentials exist yet); GitHub Actions' own built-in email notification to repo admins/watchers on a failed workflow run is the sole safety net for this case (free, already exists, no extra engineering).
- IF the script crashes anywhere after required secrets have loaded (e.g. an unforeseen bug in the dedupe/state logic, or a failure in the search step outside its normal per-query handling) THEN the system SHALL still attempt the maintainer failure-report email (P2-AC6) - this is a design gap identified during review and closed by a single top-level exception guard in the orchestrator, not by defensive handling scattered per module.

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| --- | --- | --- | --- |
| OPW-01 | P1: Daily discover, validate, publish, notify | Design | Pending |
| OPW-02 | P1: Daily discover, validate, publish, notify | Design | Pending |
| OPW-03 | P1: Daily discover, validate, publish, notify | Design | Pending |
| OPW-04 | P1: Daily discover, validate, publish, notify | Design | Pending |
| OPW-05 | P1: Daily discover, validate, publish, notify | Design | Pending |
| OPW-06 | P1: Daily discover, validate, publish, notify | Design | Pending |
| OPW-07 | P1: Daily discover, validate, publish, notify | Design | Pending |
| OPW-08 | P2: Configurable query list and recipient management | Design | Pending |
| OPW-09 | P2: Configurable query list and recipient management | Design | Pending |
| OPW-10 | P2: Failure-report email alerting | Design | Pending |
| OPW-11 | P3: Run observability | Design | Pending |

**ID format:** `OPW-[NUMBER]` (Opportunity Watch)

**Status values:** Pending → In Design → In Tasks → Implementing → Verified

**Coverage:** 11 requirement groups total (covering 30 acceptance criteria across P1-P3), 0 mapped to tasks yet, 11 unmapped ⚠️ (expected pre-Design)

---

## Success Criteria

- [ ] A scheduled run at 11:00 UTC completes end-to-end (search → validate → report → notify → commit) with zero manual intervention.
- [ ] `README.md` always reflects the current set of open opportunities after a successful run.
- [ ] A recipient receives at most one email per run, containing only opportunities not previously notified.
- [ ] Monthly OpenRouter spend (Jev classification calls) stays in the "cents, not dollars" range at the described volume.
- [ ] No recipient or maintainer email address ever appears in the git history or in the repository's (publicly-visible) workflow run logs.
- [ ] A run with any failure or exclusion produces exactly one failure-report email to every address in the maintainer alert list; a clean run produces zero.
- [ ] An unhandled exception anywhere in the pipeline (after secrets load) still results in a maintainer failure-report email, not just a silent crash caught only by GitHub's own default notifications.
