# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Session start

In every session, check the user's preferences in the mnemosyne memory server once the user's first request arrives and before acting on it. Don't act before that first request: one stored preference says to wait for it. Use `mnemosyne_recall` with `query: "preference"`, `vec_weight: 0`, `fts_weight: 0.5`, `importance_weight: 0.5`, `limit: 10`. hazard: with the default weights, a natural-language query returned 0 results even though preferences were stored. `mnemosyne_persona_list` isn't available on this server.

## Status

Planning only — no application code, dependencies, tests, or workflow YAML exist yet. The source of truth is `.specs/`:

- `.specs/features/opportunity-watch/spec.md` — requirements (OPW-01..11, EARS acceptance criteria `P1-ACn`, `P2a-ACn`, `P2b-ACn`, `P3-ACn`)
- `.specs/features/opportunity-watch/context.md` — decisions from the requirements interview (and why alternatives were rejected)
- `.specs/features/opportunity-watch/design.md` — module interfaces, data models, error-handling table, risks
- `.specs/STATE.md` — project-wide decisions (`AD-nnn`) that bind every feature

Work follows the `tlc-spec-driven` skill (Specify → Design → Tasks → Execute). Specify and Design are done; Tasks is next.

## Spec tooling

The validators must be run from the skill directory, not the repo root:

```bash
cd .claude/skills/tlc-spec-driven
python3 scripts/validate_spec.py  ../../../.specs/features/opportunity-watch/spec.md
python3 scripts/validate_tasks.py ../../../.specs/features/opportunity-watch/tasks.md
python3 scripts/validate_state.py
python3 scripts/check_commit.py --message "feat(state): add stale removal"   # Conventional Commits check
```

Run `validate_spec.py` after every spec edit. Adding an acceptance criterion in the middle of a list renumbers the rest and breaks cross-references such as `P1-AC17`, so add new ACs at the end of the list or grep for `P1-AC`/`P2a-AC`/`P2b-AC` afterwards.

## What is being built

A daily GitHub Actions job (cron `0 11 * * *` = 08:00 BRT, plus `workflow_dispatch`) in a **public** repo. It:

1. Runs `site:` queries from `config/queries.yaml` through DuckDuckGo (`ddgs`).
2. Dedupes the results by SHA-256 of the normalized URL against `data/seen.json`.
3. Checks each candidate with OpenRouter's **Jev** Decisions API. If Jev fails, it falls back to `openrouter/free` (not `openrouter/auto`, which can bill paid models).
4. Rewrites the section of `README.md` between `<!-- OPPORTUNITIES:START -->` and `<!-- OPPORTUNITIES:END -->`.
5. Emails new opportunities to `OPPORTUNITY_RECIPIENTS` and a failure report to `MAINTAINER_ALERTS` over Gmail SMTP.

Design is a small package with no framework: `config`, `search`, `state`, `validate`, `report`, `notify`, and `run` (the orchestrator). The planned dependencies are `ddgs`, `pyyaml`, and `requests`; everything else uses the stdlib. Logging is plain `print()` to stdout.

Cross-cutting invariants:

- **AD-001: never log a raw email address**, not even one address split out of a multi-address secret. GitHub masks only exact secret values, and public-repo Actions logs are readable by anyone signed in to GitHub. Log counts or non-PII identifiers instead.
- **Failure boundary has two zones.** The secret loading in `config.require_env` is deliberately left unwrapped, because without credentials no email can be sent and GitHub's built-in failure email is the only alert. Everything after that runs inside one `try/except` in `run.main()`, which makes a best-effort `send_failure_report` call and then re-raises.
- **A Jev failure is always a `FailureRecord`**, even when the free fallback recovers the candidate (P1-AC19). A candidate whose fallback also fails is left out of the report and not marked seen. It is never published unvalidated.
- **Only new ids are classified.** False positives are stored in `seen.json` with `verdict: false_positive` and are never re-classified, reported or emailed.
- **`notified` is set only after a successful send.** Each run emails every genuine, open, not-yet-notified item, so a failed or skipped send is retried automatically.
- **All emails use Bcc.** To: is the project's own sending address, so recipients never see each other.
- **Empty recipient or maintainer lists are valid.** Skip that send and don't record a failure.
- **The workflow YAML commits `seen.json` and `README.md` back to the repo, not Python.** A `concurrency` group prevents overlapping runs.
- A stale opportunity leaves the README after 3 consecutive missed runs; a run with any failed query doesn't count toward the 3. Its dedup id is kept for 30 days.

## Open risk

The exact path of the Jev Decisions API endpoint is unconfirmed: two readings of OpenRouter's docs gave `/api/alpha/decisions` and a malformed `/api/v1/api/alpha/decisions`. It is isolated in `validate.py::_call_jev`. Check the live docs (https://openrouter.ai/docs/api/api-reference/alphadecisions/submit-a-decisions-request) again before implementing it. Trust only `openrouter.ai` sources for Jev facts; search results also include SEO sites such as jevai.org and jev-ai.net, which are unreliable.
