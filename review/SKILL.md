---
name: review
description: >-
  Pre-landing PR review for TypeScript / Next.js repos. Audits plan delivery,
  runs a critical-pass checklist, dispatches Security / Testing / Performance /
  API-Contract specialists, and produces a prioritized findings report with
  file:line citations and a APPROVE / REQUEST CHANGES verdict.
  Invoke when the user says "review PR", "review this PR", or uses /review.
---

# Review — Pre-Landing PR Review

**Read-only.** Never modify code. Produce findings and suggested fixes only.

## When to invoke

User says: "review PR", "review this PR", "look at this pull request", or invokes `/review`.

## Arguments

- `/review <PR>` — full review (plan audit + critical pass + all specialists)
- `/review <PR> --quick` — critical findings only (bugs + security), skip specialists
- `/review <PR> --comment` — post findings as inline GitHub PR comments after the report

`<PR>` can be a PR number (`42`) or a full GitHub URL.

---

## Phase 0 — Setup

Fetch PR metadata:

```bash
gh pr view <PR> --json title,body,author,baseRefName,headRefName,files,additions,deletions,state
gh pr diff <PR>
```

Print one-line header: `PR #N — "<title>" by @author (+X -Y across Z files)`.

If additions + deletions > 400, print a `⚠ Large PR` warning — flag this in the report and note that review coverage may be incomplete.

If the PR is closed or merged, note it but continue — the review is still useful.

**Scope alignment check:** Read the PR title and first paragraph of the body. Does the diff actually implement what's described? If the stated scope is "fix login bug" but the diff touches unrelated modules, flag it as `SCOPE DRIFT` in the report header.

**Debug artifact scan:** Before any phase, grep the diff for `console.log`, `console.error`, `debugger`, and hardcoded test values (e.g. `@test.com`, `password123`, `localhost:`, `TODO`, `FIXME`, `HACK`) in non-test files. Collect these — report as QUALITY findings in Phase 5.

**Read CLAUDE.md** (project root) if it exists. Extract:
- `## Review Routing` table — path patterns → specialist focus areas + owner contacts
- Any repo-specific invariants listed under `## Review Rules` or similar

Keep these in mind throughout every phase.

---

## Phase 1 — Plan Audit

Look for a plan file:
1. Check the conversation context for a referenced plan/spec/task file path.
2. Check the PR description for a linked issue reference — GitHub (`#123`), or ClickUp URL. If found, note it but do not fetch it unless the tool is already available in context.
3. Check for a `PLAN.md` or `.claude/plan.md` in the repo root.

If a plan file is found, extract up to 50 actionable items (bullet points, checkboxes, numbered steps).

For each item, classify against the diff:

| Status | Meaning |
|--------|---------|
| `DONE` | Code in diff clearly implements this item |
| `PARTIAL` | Some implementation present but incomplete |
| `NOT DONE` | No corresponding code change found |
| `CHANGED` | Item was modified or replaced by a different approach |
| `UNVERIFIABLE` | External state (DB migration applied, Notion doc updated, deploy config) — cannot confirm from diff |

Print a compact plan completion table before Phase 2. Flag any `NOT DONE` items that seem load-bearing.

If no plan file is found, skip this phase silently.

---

## Phase 2 — Critical Pass

Apply this checklist to every changed file. For each finding, **quote the specific line(s)** that triggered it. Do not report a finding you cannot cite.

### TypeScript / Node

- **Type bypass** — `as any`, `as unknown as T` casts, `!` non-null assertions on values that can actually be null, `@ts-ignore` / `@ts-expect-error` outside test files
- **Unhandled rejections** — `Promise` not `.catch()`-ed, `async` function called without `await` where errors matter
- **Race conditions** — concurrent writes to shared state without a lock; `Promise.all` side effects that can partially succeed and leave data inconsistent
- **Silent error swallowing** — `catch (e) {}` or `catch (e) { console.log }` without re-throwing or returning an error the caller can act on
- **Logic errors** — off-by-one, inverted condition, unreachable branch, wrong operator

### Database

- **SQL injection** — user input in `$queryRaw`, `$executeRaw`, or any raw query string interpolation (Prisma, Knex, pg)
- **Missing transaction** — multi-step writes (insert + update, write + delete) not wrapped in a transaction
- **N+1 queries** — loop that issues a query per iteration instead of batching; Prisma relations fetched without `include`/`select`
- **Missing pagination guard** — unbounded query that can return the entire table

### Next.js / React

- **Auth bypass** — API route handler or Server Action missing auth check; Next.js middleware that skips auth for a path it shouldn't
- **Secret in client bundle** — environment variable not prefixed with `NEXT_PUBLIC_` used in a Client Component, or a `NEXT_PUBLIC_` variable that should be server-only
- **XSS** — `dangerouslySetInnerHTML` without sanitization; `innerHTML =` with unsanitized input
- **Server Action without validation** — action accepts user input without schema validation (zod, valibot, etc.)
- **Stale SSG** — `revalidate` removed or set too high for data that changes frequently; ISR pages that can serve stale security-sensitive content
- **Missing error boundary** — async Server Component without error handling, leaving the entire page broken on fetch failure

### API

- **Response shape change** — existing field removed, renamed, or type-changed without a versioning strategy
- **Required field added** — new required request field that breaks existing callers
- **Status code change** — existing endpoint returns a different HTTP status code than before

### Debug / Hygiene

- **Debug artifacts** — `console.log`, `console.error`, `debugger`, or `alert()` left in non-test files
- **Hardcoded test data** — test email addresses, fake tokens, `localhost:PORT`, or `TODO`/`FIXME`/`HACK` comments in production code paths
- **New env var without documentation** — `process.env.NEW_VAR` added in code but not added to `.env.example`, `.env.template`, or documented in the PR body

### Migrations

If the diff contains a migration file (path matches `migrations/`, `db/migrate/`, `prisma/migrations/`, or `*.sql`):
- **Destructive without guard** — `DROP TABLE`, `DELETE FROM` without a `WHERE` clause, or `TRUNCATE` not wrapped in a transaction with an explicit rollback plan
- **Irreversible** — migration has no `down` / rollback counterpart and removes or renames a column; flag if the PR doesn't acknowledge this

### Confidence gate

Only report findings you're ≥ 7/10 confident are real. Label 5–6/10 as `(TENTATIVE)`. Suppress anything below 5.

---

## Phase 3 — Specialist Dispatch

Skip all specialists for `--quick`.

Dispatch all four specialists **in parallel** using the Agent tool — one agent per specialist. Each agent receives the full diff as context and returns its findings. Collect results before writing the report. This cuts wall-clock time to the slowest specialist, not the sum.

### Security Specialist

Focus on the changed code only.

- **Hardcoded credentials** — API keys, tokens, passwords, private keys in diff
- **IDOR** — new endpoints that return or modify data keyed by a user-supplied ID without verifying ownership
- **CSRF** — state-mutating endpoints (POST/PUT/DELETE) missing CSRF protection where relevant
- **Injection paths** — user input flowing into shell commands (`exec`, `spawn`, `child_process`), template engines, or eval
- **Sensitive data in logs** — passwords, tokens, PII printed to stdout/logs/error messages
- **Webhook signature bypass** — webhook receiver that doesn't verify the HMAC header (Stripe, GitHub, Slack patterns)

For each hit, trace the full code path: confirm user input actually reaches the sink before reporting.

### Testing Specialist

- **Missing happy-path test** — new function or endpoint with no corresponding test
- **Missing error-path test** — error branches (DB failure, auth failure, invalid input) with no test coverage
- **Test that doesn't assert** — test that calls code but only checks it doesn't throw, without asserting the actual output
- **Mocked-away reality** — test mocks the thing it's supposed to be testing (e.g., mocks the DB and then asserts the mock was called)
- **Snapshot drift** — snapshot file updated in the diff without a review comment explaining what changed visually; silent snapshot updates mask regressions
- **Regression gap** — changed behavior that existing tests no longer cover (the test still passes but now tests the old behavior against new code)

Only flag coverage gaps that correspond to real risk — skip trivial getters, boilerplate, and pure type definitions.

### Performance Specialist

Focus on backend, data-access, and frontend bundle code.

**Backend / data access:**
- **N+1** (second pass) — loop over a result set that issues a DB call per item
- **Missing index hint** — query on a non-indexed column in a table likely to grow large; flag only if the query is on a hot path
- **Unbounded fetch** — API call or DB query with no limit that could return arbitrarily large payloads
- **Synchronous heavy work on the request thread** — CPU-bound operation (parsing, encoding, hashing large payloads) blocking the event loop
- **Cache invalidation gap** — data updated in one place but stale in a cache that isn't invalidated
- **Waterfall fetch** — sequential `await` calls that could be parallelized with `Promise.all`

**Frontend / bundle:**
- **Barrel import of large library** — `import * as Foo from 'large-lib'` or `import { everything } from 'lib'` where tree-shaking is blocked; flag if the library is known-large (lodash, moment, antd, etc.)
- **New heavy dependency** — `package.json` change adds a dependency >50 KB gzipped with a lighter alternative available
- **Re-render trigger** — inline object/function created in render passed as a prop to a memoized component, defeating `React.memo`; `useEffect` dependency array includes an unstable reference
- **Missing `loading` / `Suspense` boundary** — new data-fetching component has no loading state, causing layout shift or blank content on slow connections

### API Contracts Specialist

- **Breaking response change** — field removed, renamed, or type-narrowed in a response that existing clients depend on
- **New required request field** — existing callers that don't send this field will now fail
- **Undocumented status code** — endpoint now returns a status code not in its spec/types
- **Pagination contract change** — cursor/offset scheme changed in a way that breaks clients mid-pagination
- **Auth requirement added** — previously public endpoint now requires auth (silent breaking change for integrations)

If the repo's `CLAUDE.md` routing table lists an owner for the changed path, note their handle next to findings in that area.

---

## Phase 4 — False Positive Filter

Before writing any finding to the report, apply this gate:

**Auto-discard:**
- Style issues (formatting, naming conventions, import order)
- "Could be more idiomatic" observations with no correctness or security impact
- Performance micro-optimizations not on a measured hot path
- Issues already acknowledged in the PR description as known limitations
- Issues in test files that don't affect production behavior
- CVEs or dependency alerts where the vulnerable code path is not called

**Confidence floor:** Suppress anything below 5/10. Label 5–6/10 as `(TENTATIVE)`.

---

## Phase 5 — Report

### Summary table

```
PR REVIEW — #N "<title>"
Author: @author | +X -Y across Z files
Plan: M/N items DONE  [or "No plan file found"]
Scope: ALIGNED  [or "SCOPE DRIFT — diff touches X but PR describes Y"]

#   Severity    Conf    Effort   Specialist          Finding                                File:Line
──  ─────────   ────    ──────   ──────────          ───────                                ─────────
1   BUG         9/10    Medium   Critical Pass       Null deref when session expires        src/auth.ts:42
2   SECURITY    8/10    Quick    Security            Token printed on failed login          src/api/login.ts:88
3   TESTS       7/10    Quick    Testing             No test for invalid-input path         src/api/orders.ts:120
4   QUALITY     6/10    Hard     Performance (TENT.) Waterfall await in hot endpoint        src/api/search.ts:55

VERDICT: REQUEST CHANGES  (1 bug, 1 security, 1 test gap)
```

**Effort key:** `Quick` = <15 min / a few lines; `Medium` = 15–60 min / meaningful refactor; `Hard` = >1 hour / design change needed.

### Finding format

```
## Finding N: [Title] — [file:line]

* **Severity:** BUG | SECURITY | QUALITY | TESTS | PERFORMANCE
* **Confidence:** N/10
* **Effort:** Quick | Medium | Hard
* **Specialist:** Critical Pass | Security | Testing | Performance | API Contracts
* **Summary:** One sentence — what is wrong and why it matters
* **Scenario:** Concrete situation where this causes a problem
* **Suggested fix:** Specific change, with a short code example if helpful
* **Owner:** @handle  [only if CLAUDE.md routing table lists one for this path]
```

### Severity guide

| Level | Meaning |
|-------|---------|
| BUG | Incorrect behavior in production; blocks or degrades users |
| SECURITY | Exploitable vulnerability or data exposure |
| PERFORMANCE | Measurable latency or resource problem on a real path |
| TESTS | Missing coverage for a real risk |
| QUALITY | Will slow down future changes or mislead the next reader |

### Verdict

- **APPROVE** — no BUG or SECURITY findings; only QUALITY / TESTS notes (or none)
- **APPROVE WITH SUGGESTIONS** — no blockers; QUALITY / TESTS / PERFORMANCE findings worth addressing
- **REQUEST CHANGES** — one or more BUG or SECURITY findings

If zero findings: print `No issues found above confidence threshold.` — do not invent problems.

---

## Phase 6 — Post Comments (--comment only)

If `--comment` was passed, post each finding as an inline review comment:

```bash
# Top-level review comment
gh pr review <PR> --comment --body "<summary table>"

# Inline comment on a specific line
gh api repos/{owner}/{repo}/pulls/{PR}/comments \
  --method POST \
  --field body="<finding text>" \
  --field commit_id="$(gh pr view <PR> --json headRefOid -q .headRefOid)" \
  --field path="<file>" \
  --field line=<line>
```

After posting: print `Posted N inline comments on PR #<PR>.`

---

## Self-check before returning

- [ ] Every finding quotes the specific file and line that triggered it
- [ ] Every BUG finding has a concrete scenario where wrong behavior occurs
- [ ] No style-only findings in the report
- [ ] CLAUDE.md routing table was read and owner handles attached where applicable
- [ ] Verdict matches the findings (REQUEST CHANGES if any BUG or SECURITY present)
- [ ] If `--comment` was passed, comments were actually posted
- [ ] Confidence gate applied — nothing below 5/10 in the report
- [ ] Debug artifact scan completed — `console.log` / `debugger` / hardcoded test data checked in non-test files
- [ ] Scope alignment line present in report header (ALIGNED or SCOPE DRIFT)
- [ ] Large PR warning printed if additions + deletions > 400
- [ ] Every finding includes an Effort estimate (Quick / Medium / Hard)
