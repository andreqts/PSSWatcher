# STATE

## Decisions

### AD-001
- **Decision**: Workflow/job logs may never contain a raw recipient or maintainer email address (or any other secret value/PII), even partially. Log counts and identifiers only (e.g. "sent to 3 recipients", not the addresses themselves).
- **Reason**: This repo is public, and GitHub's own docs confirm public-repo Actions run logs are visible to any signed-in GitHub user. GitHub auto-masks an *exact* secret string in logs, but not a sub-string extracted from it (e.g. one address split out of a multi-address secret) - so naive logging would leak PII into a permanently public place.
- **Trade-off**: Slightly less debuggable logs (can't eyeball which address failed) - mitigated by logging a stable non-PII identifier (e.g. recipient index or a truncated hash) instead of the raw address when per-address detail is needed.
- **Scope**: Any feature/component that logs to stdout/stderr in a GitHub Actions job on this repo.
- **Date**: 2026-09-22
- **Status**: active

## Handoff

(none - no session pause yet)
