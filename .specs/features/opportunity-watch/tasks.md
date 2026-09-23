# Opportunity Watch Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Spec**: `.specs/features/opportunity-watch/spec.md`
**Design**: `.specs/features/opportunity-watch/design.md`
**Status**: Draft

**Layout (decided 2026-09-22)**: every module lives in `src/opportunity_watch/` (`design.md` component locations match); tests live in `tests/test_<module>.py`; dependencies and tool config live in `pyproject.toml`, managed with `uv`.

**Commits**: one Conventional Commit per task (checked with `check_commit.py`). Per the user's standing preference, confirm at the start of Execute that approving this file counts as the explicit request to make these per-task commits.

---

## Test Coverage Matrix

> Generated from project guidelines and spec - confirm before Execute. Guidelines found: `CLAUDE.md`, `.specs/STATE.md` (AD-001: no raw email address in any log output). No testing guideline or existing tests in the repo - strong defaults applied. Test types and commands chosen by the user on 2026-09-22: pytest with faked I/O, ruff for lint/format, no real network calls in the test suite.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| ---------- | ------------------ | -------------------- | ---------------- | ----------- |
| Domain logic (`state`, `report`, `validate.classify`, `config` parsing) | unit | All branches; 1:1 to spec ACs; every listed edge case has a test | `tests/test_<module>.py` | `uv run pytest -q` |
| External adapters (`search`, `validate._call_jev` / `_call_free_fallback`, `notify` SMTP) | unit, I/O faked via monkeypatch | Happy path + every failure path the design's error table names; request/response shape asserted against the verified API contract; AD-001 asserted (no address in captured stdout/stderr) | `tests/test_<module>.py` | `uv run pytest -q` |
| Orchestrator (`run.main`) | integration, all I/O faked, real files in `tmp_path` | Every branch of the design diagram: clean run, per-query failure, Jev degraded, both validators fail, email failure, empty lists, unhandled exception after secrets, missing secret; exit codes and summary line asserted | `tests/test_run.py` | `uv run pytest -q` |
| Entity / scaffold / config data / workflow YAML | none | - (build gate only) | - | build gate only |
| Live end-to-end | manual e2e via `workflow_dispatch` | The spec's P1 and P2b Independent Tests, with evidence captured | GitHub Actions run log | manual |

## Gate Check Commands

> Generated from the chosen tooling - confirm before Execute.

| Gate Level | When to Use | Command |
| ---------- | ----------- | ------- |
| Scaffold | Tasks before any test exists (pytest exits 5 on "no tests collected") | `uv sync && uv run ruff check . && uv run ruff format --check . && uv run python -c "import opportunity_watch"` |
| Quick | After tasks with unit tests only | `uv run pytest -q` |
| Full | After tasks with integration tests | `uv run pytest -q` (unit + integration share one suite) |
| Build | After phase completion or config/entity-only tasks | `uv run ruff check . && uv run ruff format --check . && uv run pytest -q` |
| Manual | Live end-to-end task only | `workflow_dispatch` run on GitHub + evidence recorded in the task |

---

## Execution Plan

Phases are ordered and run sequentially - each phase completes before the next begins, and tasks within a phase execute in order.

### Phase 1: Foundation

```
T1 → T2
```

### Phase 2: Configuration

```
T3 → T4
```

### Phase 3: State

```
T5 → T6 → T7 → T8
```

### Phase 4: External adapters and rendering

```
T9
T10 → T11 → T12
T13
T14 → T15
```

### Phase 5: Orchestration and delivery

```
T16 → T17 → T18
```

### Phase 6: Live verification

```
T19
```

---

## Task Breakdown

### Phase 1: Foundation

#### T1: Create the project scaffold

**What**: `pyproject.toml` for package `opportunity_watch` (Python >=3.12, src layout; runtime deps `ddgs`, `pyyaml`, `requests`; dev group `pytest`, `ruff`), an empty `src/opportunity_watch/__init__.py`, and a `.gitignore` for `.venv/`, `__pycache__/`, `.pytest_cache/`, `.ruff_cache/`; commit `uv.lock`.
**Where**: `pyproject.toml` (plus the empty package `__init__` and `.gitignore` - one cohesive scaffold)
**Depends on**: None
**Reuses**: nothing (first code in the repo)
**Requirement**: OPW-01 (enabler for all)

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] `uv sync` succeeds and creates `uv.lock`
- [ ] Scaffold gate passes

**Tests**: none (scaffold - matrix says build gate only)
**Gate**: scaffold
**Commit**: `build: add pyproject, package skeleton and gitignore`

---

#### T2: Define the data models

**What**: `RawResult`, `Candidate` (with `verdict` and `notified`), `ValidationResult`, `FailureRecord` (with `excluded: bool = False`), `RunSummary`, `NotifyResult` as dataclasses exactly as in `design.md` Data Models.
**Where**: `src/opportunity_watch/models.py`
**Depends on**: T1
**Reuses**: `design.md` Data Models section
**Requirement**: OPW-02, OPW-03, OPW-10

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Every field in `design.md` Data Models is present with the same name and type
- [ ] Scaffold gate passes

**Tests**: none (entity - matrix says build gate only)
**Gate**: scaffold
**Commit**: `feat(models): add opportunity watch data models`

---

### Phase 2: Configuration

#### T3: Implement config loading

**What**: `require_env(name)` raising `ConfigError`, `load_address_list(env_var)` (newline-separated, blanks stripped, `[]` when unset, never raises), `load_queries(path)` reading `queries:` from YAML, and `jev_model()` returning `JEV_MODEL` or the default `typesafe/jev-latest`.
**Where**: `src/opportunity_watch/config.py`
**Depends on**: T2
**Reuses**: `pyyaml`
**Requirement**: OPW-08, OPW-09, OPW-02

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests cover: missing required secret raises `ConfigError` naming the variable but never its value; address list with blank lines/whitespace; unset address list returns `[]`; queries file parsed; malformed YAML raises; `JEV_MODEL` override and default (P2a-AC1, P2a-AC2, P2b-AC4)
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(config): load secrets, address lists, queries and jev model`

---

#### T4: Add the query list config file

**What**: `config/queries.yaml` with the 8 UNILA/IFPR/UNIOESTE queries from `design.md`, and a unit test that loads the shipped file through `load_queries`.
**Where**: `config/queries.yaml`
**Depends on**: T3
**Reuses**: `config.load_queries` (T3)
**Requirement**: OPW-08

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Test asserts the shipped file parses, is non-empty and every query starts with `site:` (P2a-AC1, P2a-AC3)
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(config): add initial site query list`

---

### Phase 3: State

#### T5: Implement URL normalization and opportunity id

**What**: `normalize_url(url)` (lowercase scheme/host, drop fragment, drop `utm_*` params, drop trailing `/`) and `opportunity_id(url)` = SHA-256 hex of the normalized URL.
**Where**: `src/opportunity_watch/state.py`
**Depends on**: T2
**Reuses**: stdlib `hashlib`, `urllib.parse`
**Requirement**: OPW-03

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests: equivalent URLs (case, fragment, tracking params, trailing slash) give the same id; different paths give different ids (P1-AC8, edge case "same URL from two queries")
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(state): add url normalization and opportunity id`

---

#### T6: Implement SeenStore load, save and diff_new

**What**: `load_seen(path)` (missing file = empty store), `SeenStore.save(path)` writing the `design.md` JSON shape, `SeenStore.diff_new(ids)` returning ids not in the store.
**Where**: `src/opportunity_watch/state.py`
**Depends on**: T5
**Reuses**: stdlib `json`
**Requirement**: OPW-03, OPW-06

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests: first-run missing file gives an empty store and every id is unseen (edge case "first-ever run"); save/load round-trip preserves every field; `diff_new` excludes both genuine and false-positive entries (P1-AC4)
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(state): load, save and diff the seen store`

---

#### T7: Implement SeenStore.apply_run

**What**: `apply_run(found_ids, new_candidates, today, count_misses)` (signature in `design.md`) inserting new candidates with their verdicts, and for every found id updating `last_seen`, `miss_count`, stale removal after 3 counted misses, 30-day purge, and reappearance; returns the open genuine set and purged ids.
**Where**: `src/opportunity_watch/state.py`
**Depends on**: T6
**Reuses**: `SeenStore` (T6)
**Requirement**: OPW-02, OPW-05, OPW-06

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests, one per rule: false positive stored and never in the open set (P1-AC5); absent 3 counted runs leaves the open set (P1-AC13); `count_misses=False` never increments `miss_count` (P1-AC21); purge only after 30 days since `last_seen` (P1-AC14); reappearing id returns to the open set with `miss_count` 0 and `notified` unchanged (P1-AC22); an already-seen id in `found_ids` keeps its stored `verdict` and `notified` (a stored `false_positive` is never overwritten)
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(state): apply run results with stale removal and purge`

---

#### T8: Implement pending_notification and mark_notified

**What**: `pending_notification()` returning genuine, open, not-notified entries; `mark_notified(ids)`.
**Where**: `src/opportunity_watch/state.py`
**Depends on**: T7
**Reuses**: `SeenStore` (T6, T7)
**Requirement**: OPW-03, OPW-04

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests: pending excludes false positives, stale entries and notified entries (P1-AC9); after `mark_notified` the ids are no longer pending (P1-AC23); ids not marked stay pending on the next load (retry after failed send)
- [ ] Build gate passes (end of phase); test count only grows

**Tests**: unit
**Gate**: build
**Commit**: `feat(state): track which opportunities were notified`

---

### Phase 4: External adapters and rendering

#### T9: Implement run_searches

**What**: `run_searches(queries)` calling `DDGS().text(query, max_results=N)` per query; returns `RawResult`s and `FailureRecord(type="search")` per failed query; drops results missing title or URL.
**Where**: `src/opportunity_watch/search.py`
**Depends on**: T2
**Reuses**: `ddgs`, `models` (T2)
**Requirement**: OPW-01

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests with a fake `DDGS`: fields extracted (P1-AC2); result missing title or URL dropped, rest kept (P1-AC3); one query raising produces one `FailureRecord` and the other queries still run (edge case "backend errors"); zero results is not a failure (edge case)
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(search): run site queries with per-query failure isolation`

---

#### T10: Implement the Jev call

**What**: First re-verify the live Decisions API endpoint, request/response schema and the `typesafe/jev-latest` model id on `openrouter.ai` only (record the URLs checked and the date in `design.md` Risks); then implement `_call_jev(candidate)` sending the exact P1-AC20 question, model from `config.jev_model()`, raising `JevCallError` on HTTP error, timeout or unparseable answer, and mapping the probability against the 0.5 threshold.
**Where**: `src/opportunity_watch/validate.py`
**Depends on**: T3
**Reuses**: `requests`, `config.jev_model` (T3)
**Requirement**: OPW-02

**Tools**:

- MCP: NONE
- Skill: NONE (WebFetch for the endpoint re-verification, `openrouter.ai` domain only)

**Done when**:

- [ ] Endpoint, schema and model id verified against live `openrouter.ai` docs; the evidence (URL + date) is written into `design.md` Risks, replacing the "unconfirmed" note
- [ ] Unit tests with a fake HTTP layer: request body contains the P1-AC20 question verbatim and the configured model; yes above threshold is genuine, below is false positive; HTTP error, timeout and malformed JSON each raise `JevCallError` (P1-AC6 trigger conditions)
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(validate): classify candidates with the jev decisions api`

---

#### T11: Implement the free-tier fallback call

**What**: `_call_free_fallback(candidate)` posting to `/api/v1/chat/completions` with `"model": "openrouter/free"` and the same P1-AC20 question, parsing a yes/no answer, raising `ValidationFailed` on any failure.
**Where**: `src/opportunity_watch/validate.py`
**Depends on**: T10
**Reuses**: HTTP helper and question constant from T10
**Requirement**: OPW-02

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests: model is exactly `openrouter/free` and never `openrouter/auto` or a paid id (P1-AC6); yes/no parsed; error, timeout and unusable answer raise `ValidationFailed` (P1-AC7)
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(validate): add free-tier fallback classification`

---

#### T12: Implement classify

**What**: `classify(candidate) -> tuple[ValidationResult, list[FailureRecord]]` (signature in `design.md`): Jev first; on `JevCallError` the fallback plus a degraded `FailureRecord(excluded=False)`; when both fail, verdict `failed` plus `FailureRecord(excluded=True)`.
**Where**: `src/opportunity_watch/validate.py`
**Depends on**: T11
**Reuses**: `_call_jev` (T10), `_call_free_fallback` (T11)
**Requirement**: OPW-02

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests for all five paths: Jev genuine; Jev false positive; Jev fails and fallback says genuine (verdict `genuine`, `method="free_fallback"`, degraded failure recorded, P1-AC19); Jev fails and fallback says false positive (verdict `false_positive`, degraded failure still recorded); both fail (verdict `failed`, excluded failure recorded, P1-AC7)
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(validate): chain jev and fallback into classify`

---

#### T13: Implement the README report

**What**: `render_section(open_opportunities)` and `update_readme(path, rendered)` replacing only the text between `<!-- OPPORTUNITIES:START -->` and `<!-- OPPORTUNITIES:END -->` and returning whether the file changed.
**Where**: `src/opportunity_watch/report.py`
**Depends on**: T2
**Reuses**: `models` (T2)
**Requirement**: OPW-05

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests: content outside the markers is byte-identical; an empty set renders an explicit "no open opportunities" line; unchanged input returns `False` (P1-AC12, P1-AC15); missing markers raise a clear error
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(report): render open opportunities into the readme`

---

#### T14: Implement the SMTP send helper

**What**: `_send(subject, body, bcc_list)` over Gmail SMTP (`smtp.gmail.com:465`, SSL) with `GMAIL_ADDRESS` / `GMAIL_APP_PASSWORD`, To: the sender address, every recipient only in Bcc.
**Where**: `src/opportunity_watch/notify.py`
**Depends on**: T3
**Reuses**: stdlib `smtplib`, `email.message`
**Requirement**: OPW-04, OPW-10

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests with a fake `SMTP_SSL`: To header is the sender; no recipient address appears in any header; every recipient is in the envelope recipients (P1-AC10, P2b-AC8); an SMTP exception propagates as a typed error; no address appears in captured stdout/stderr (AD-001)
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `feat(notify): add bcc smtp send helper`

---

#### T15: Implement the two notification emails

**What**: `send_new_opportunities(items, recipients)` and `send_failure_report(failures, maintainers)` returning a `NotifyResult` (counts only), skipping on empty lists or no items, and catching a send error into `NotifyResult.failure` - a `FailureRecord(type="email")` whose target is a count, never an address.
**Where**: `src/opportunity_watch/notify.py`
**Depends on**: T14
**Reuses**: `_send` (T14)
**Requirement**: OPW-04, OPW-10, OPW-11

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit tests: one email listing only the given items (P1-AC10); no items or empty list means no send and no failure (P1-AC11, edge cases); failure report lists type, target and message for each failure (P2b-AC3); send error is returned as `NotifyResult.failure` (not raised) with no address in it (P1-AC16, P3-AC1, AD-001)
- [ ] Build gate passes (end of phase); test count only grows

**Tests**: unit
**Gate**: build
**Commit**: `feat(notify): send new-opportunity and failure-report emails`

---

### Phase 5: Orchestration and delivery

#### T16: Create the README with report markers

**What**: `README.md` with a short project description and an empty `<!-- OPPORTUNITIES:START -->` / `<!-- OPPORTUNITIES:END -->` section for `report.update_readme`.
**Where**: `README.md`
**Depends on**: T13
**Reuses**: marker constants from `report.py` (T13)
**Requirement**: OPW-05

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Unit test in `tests/test_report.py` runs `update_readme` on a copy of the real `README.md`: both markers appear exactly once and content outside them is unchanged
- [ ] Quick gate passes; test count only grows

**Tests**: unit
**Gate**: quick
**Commit**: `docs: add readme with opportunity report section`

---

#### T17: Implement the orchestrator

**What**: `run.main()` following the `design.md` diagram: secrets loaded outside the try; everything after inside one `try/except` that best-effort sends the failure report, prints the partial summary and re-raises; `count_misses` false when any search failed; `seen.json` saved after the send so `notified` persists; failure report checked last; `RunSummary` printed with counts only. Entry point `python -m opportunity_watch.run` through `if __name__ == "__main__": raise SystemExit(main())`, so the returned code becomes the process exit code. Builds `found_ids` from all deduped results and passes only freshly classified entries as `new_candidates`; appends each `NotifyResult.failure` to the run's failures.
**Where**: `src/opportunity_watch/run.py`
**Depends on**: T16
**Reuses**: every module from T3-T15
**Requirement**: OPW-07, OPW-10, OPW-11

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] Integration tests with all I/O faked and real files in `tmp_path`, one per diagram branch: clean run with new items (README, `seen.json`, one email, exit 0); no pending (not yet notified) items gives no email (P1-AC11); a previous send failed and nothing is newly discovered today, so the pending items are emailed and then marked `notified` (P1-AC9, P1-AC23); per-query failure (failure report sent, no miss counted, P1-AC21); Jev degraded (item reported and failure report sent, P1-AC19); degraded false positive (stored as `false_positive`, not reported, failure report still sent); both validators fail (item excluded, not stored, retried next run, P1-AC7); new-opportunity email fails (items stay un-notified, failure report sent, P1-AC23, P2b-AC1); failure plus new items gives two separate emails (P2b-AC5); clean run sends no failure report (P2b-AC2); exception after secrets gives a failure report, a summary line and a non-zero exit (P2b-AC6, P3-AC2); missing secret exits non-zero with no email attempt (P2b-AC7); summary line has every P1-AC18 count; no address in captured output (AD-001); running the module in a subprocess returns the expected exit code
- [ ] Full gate passes; test count only grows

**Tests**: integration
**Gate**: full
**Commit**: `feat(run): orchestrate the daily opportunity watch`

---

#### T18: Add the GitHub Actions workflow

**What**: Workflow with `schedule: cron "0 11 * * *"` and `workflow_dispatch`, a `concurrency` group with `cancel-in-progress: false`, `permissions: contents: write`, `uv` setup, secrets as env vars (`OPENROUTER_API_KEY`, `GMAIL_ADDRESS`, `GMAIL_APP_PASSWORD`, `OPPORTUNITY_RECIPIENTS`, `MAINTAINER_ALERTS`), `JEV_MODEL` from repository variables, `python -m opportunity_watch.run`, then commit and push `data/seen.json` and `README.md` only if they changed.
**Where**: `.github/workflows/opportunity-watch.yml`
**Depends on**: T17
**Reuses**: nothing
**Requirement**: OPW-01, OPW-06, OPW-07

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] YAML parses (`uv run python -c "import yaml; yaml.safe_load(open('.github/workflows/opportunity-watch.yml'))"`) and contains the cron, dispatch, concurrency and commit-if-changed steps (P1-AC1, P1-AC15, P1-AC17)
- [ ] No step echoes a secret or an address (AD-001)
- [ ] Build gate passes

**Tests**: none (workflow YAML - matrix says build gate only; its behavior is verified live in T19)
**Gate**: build
**Commit**: `ci: add daily opportunity watch workflow`

---

### Phase 6: Live verification

#### T19: Run the live end-to-end check

**What**: With the user's go-ahead to push and with the dedicated Gmail account and secrets created by the user, trigger `workflow_dispatch` and run the spec's P1 and P2b Independent Tests; record run URLs and outcomes in `validation.md`.
**Where**: `.specs/features/opportunity-watch/validation.md`
**Depends on**: T18
**Reuses**: spec Independent Tests
**Requirement**: OPW-01 to OPW-11

**Tools**:

- MCP: NONE
- Skill: NONE (`gh` CLI to trigger the run and read logs)

**Done when**:

- [ ] **Blocked until the user**: creates the dedicated Gmail account and App Password, sets the five secrets, and approves `git push`
- [ ] A dispatched run commits `seen.json` and `README.md` and sends the new-opportunity email (P1 Independent Test)
- [ ] A run with a forced validation failure sends one failure report to each maintainer address (P2b Independent Test)
- [ ] The public run log contains no email address (AD-001)

**Tests**: e2e (manual)
**Gate**: manual
**Commit**: `test(e2e): record live workflow verification`

---

## Phase Execution Map

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6

Phase 1:  T1 ──→ T2
Phase 2:  T3 ──→ T4
Phase 3:  T5 ──→ T6 ──→ T7 ──→ T8
Phase 4:  T9
          T10 ──→ T11 ──→ T12
          T13
          T14 ──→ T15
Phase 5:  T16 ──→ T17 ──→ T18
Phase 6:  T19
```

Execution is strictly sequential - a single agent (or batch worker) works one task at a time, in order.
