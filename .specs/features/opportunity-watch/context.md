# Opportunity Watch Context

**Gathered:** 2026-09-22
**Spec:** `.specs/features/opportunity-watch/spec.md`
**Status:** Ready for design

---

## Feature Boundary

A daily scheduled job that searches a fixed list of university sites for matching academic opportunities, filters false positives with an LLM check, publishes the current open set to `README.md`, and emails a manually-managed recipient list about only the newly-found ones. No self-serve subscription in v1.

---

## Implementation Decisions

### Search backend

- DuckDuckGo via the `ddgs` Python package (free, no API key), running the existing `site:domain "query text"` list.
- Google was ruled out - scraping it gets CAPTCHA'd/rate-limited without a paid API.

### Dedup / state persistence

- `data/seen.json`, committed back to the repo by the workflow itself each run.
- Rejected an external DB (Postgres/Redis) as unnecessary cost/ops for this volume.

### Report publishing

- Rewrite a marked section of `README.md` in place; GitHub renders it automatically, no extra infra.
- GitHub Pages explicitly deferred as a later upgrade, not built now (YAGNI).

### Email delivery

- Gmail SMTP via Python's stdlib `smtplib`, using a **new, dedicated free Gmail account** (not the user's personal one) with an App Password.
- Rejected a transactional email API (Resend/SendGrid) - new account and dependency for no benefit at this volume.

### Recipient list & privacy

- Repo is **public** by intent (the report must be publicly visible).
- Recipient emails are **not** committed to the repo (would leak into permanent git history). They live in a single GitHub Actions secret, newline-separated, edited manually by the repo admin.
- Self-serve signup (a button/form for people to subscribe themselves) was requested but explicitly deferred - see Deferred Ideas.

### Validation check: Jev, not a general LLM

- User corrected an earlier assumption: the false-positive check must use **Jev**, OpenRouter's TypeSafe decision model - not a general chat-completion LLM prompt.
- Verified live (not from training knowledge) against OpenRouter's own docs on 2026-09-22: `openrouter.ai/typesafe` (model catalog page) and `openrouter.ai/docs/cookbook/evaluate-and-optimize/jev-verified-cascade` (usage pattern). Jev is served through the same OpenRouter API and API key already planned - no new account or credential needed.
- Jev returns a typed verdict (e.g. `supported`/`unsupported`/`declined`) plus a confidence score rather than free-form text - a better fit for a true/false-positive gate than prompting a general LLM.
- Caveat: the specific serving surface (OpenRouter's "Decisions" API) is explicitly labeled **alpha** by OpenRouter. Design includes a fallback to a standard OpenRouter chat-completion request if Jev's endpoint proves unstable - but per explicit user instruction, that fallback is restricted to free models only. If the free-tier fallback also fails, the candidate is excluded and the failure logged, same as any other validation failure (OPW P1-AC7) - it is never published unvalidated.
- User initially proposed `openrouter/auto` (OpenRouter's general Auto Router) restricted to free models. Verified live against OpenRouter's own docs (`openrouter.ai/docs/faq`, `openrouter.ai/openrouter/auto`) on 2026-09-22: `openrouter/auto` has no free-only restriction mechanism and is documented to route to (and bill) paid models - combining "auto" with "free-only" isn't a supported combination. The correct mechanism is OpenRouter's separate, purpose-built **`openrouter/free`** (Free Models Router), which auto-selects among free-tier models only. Fallback uses `openrouter/free`, not `openrouter/auto`.
- Exact model version (`typesafe/jev-1.13` vs `typesafe/jev-latest`) and current pricing: deferred to Design time, not fabricated now.
- Runs against paid OpenRouter credits rather than a free-tier model - user prioritizes reliability over $0 cost.

### Schedule

- Daily cron at 08:00 América/São Paulo. User initially wrote 8:00 PM, then corrected to 8:00 AM. Brazil has had no DST since 2019, so BRT is a fixed UTC-3 - the workflow cron uses **11:00 UTC**.

### Maintainer failure alerts

- User's stated fear: an overlooked failure (e.g. Jev + free-fallback both failing for one candidate) silently drops a real opportunity, with no one noticing.
- GitHub Actions already emails repo admins/watchers for free when a workflow run hard-fails (uncaught exception, non-zero exit) - that part needs no new engineering.
- The gap being closed here is a run that **succeeds overall** while quietly excluding a candidate. New requirement: a daily failure-report email, sent only when a run recorded at least one failure/exclusion, to every address in a maintainer alert list (one or more, newline-separated) stored in its own GitHub Actions secret (kept separate from the interested-recipient list so the two audiences never mix).
- Both lists follow the same shape: a newline-separated set of addresses in a dedicated GitHub Actions secret. There are always exactly two such secrets/lists - opportunity recipients and maintainer alerts - never combined into one.
- Reuses the same Gmail SMTP mechanism already built for P1 - no new dependency or account.

### Agent's Discretion

- Opportunity dedup key: SHA-256 of the normalized result URL (no natural ID exists in scraped search results).
- Validation-failure handling: an OpenRouter error on a candidate excludes it from that run rather than publishing it unvalidated.
- Stale-opportunity handling: drop from the README after 3 consecutive runs missing from search results; keep the dedup ID for 30 days so a transient search miss doesn't cause a duplicate "new" notification.
- Concurrency: a GitHub Actions `concurrency` group prevents overlapping runs corrupting `seen.json`/README on a manual re-trigger.

All four are flagged in the spec's Assumptions table as agent defaults - revisit if any don't match what the user actually wants once they're visible in a real run.

---

## Specific References

- Example search queries the user already has in mind:
  - `site:unila.edu.br "professor substituto"`
  - `site:utfpr.edu.br "professor substituto"`
  - `site:ifpr.edu.br "processo seletivo simplificado"`
  - `site:ufpr.br "professor substituto"`
  These seed `config/queries.yaml` (P2) rather than being hardcoded.
- Subject areas: electrical, electronic, energy, computer engineering, software engineering, computer science, or related fields.
- Location filter: Foz do Iguaçu, Paraná, Brazil.
- Jev/TypeSafe verification sources checked live: [OpenRouter TypeSafe model page](https://openrouter.ai/typesafe), [OpenRouter Jev cookbook](https://openrouter.ai/docs/cookbook/evaluate-and-optimize/jev-verified-cascade). Other search hits (jevai.org, jev-ai.net, third-party blog posts with hyperbolic performance claims) were not used as sources - only OpenRouter's own domain was treated as authoritative.

---

## Deferred Ideas

- **Self-serve subscription.** User liked the Cloudflare Worker + Turnstile-captcha option (verify a submission server-side, write straight to the recipient store) but explicitly asked to keep it in the backlog for a future iteration. For v1, recipients are added manually to the GitHub Actions secret. Revisit as its own feature once the core watcher is running.
