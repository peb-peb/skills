---
name: review
description: >-
  Pre-landing PR review for the oscode monorepo and any TypeScript repo.
  Auto-detects project type (NestJS / Next.js / Expo RN) and applies the
  matching ruleset. Audits plan delivery, runs a critical-pass checklist,
  dispatches Security / Testing / Performance / API-Contract / Architecture
  specialists, and produces a prioritized findings report with file:line
  citations and a APPROVE / REQUEST CHANGES verdict.
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
gh pr view <PR> --json title,body,author,baseRefName,headRefName,files,additions,deletions,state,headRepository
gh pr diff <PR>
```

Print one-line header: `PR #N — "<title>" by @author (+X -Y across Z files)`.

If additions + deletions > 400, print a `⚠ Large PR` warning — flag this in the report and note that review coverage may be incomplete.

If the PR is closed or merged, note it but continue.

**Scope alignment check:** Read the PR title and first paragraph of the body. Does the diff actually implement what's described? If the stated scope is "fix login bug" but the diff touches unrelated modules, flag it as `SCOPE DRIFT` in the report header.

**Debug artifact scan:** Grep the diff for `console.log`, `console.error`, `debugger`, and hardcoded test values (e.g. `@test.com`, `password123`, `localhost:`, `TODO`, `FIXME`, `HACK`) in non-test files. Collect — report as QUALITY findings in Phase 5.

**Read CLAUDE.md** (project root) if it exists. Extract:
- `## Review Routing` table — path patterns → specialist focus areas + owner contacts
- Any repo-specific invariants listed under `## Review Rules` or similar

### Project Fingerprint

Detect the project type from the repo name or changed file paths. Set `PROJECT_TYPE` to one of:

| Repo name(s) | File path signals | `PROJECT_TYPE` |
|---|---|---|
| `oscode-be`, `urbuddy-be`, `oscode-search-be` | `src/modules/`, `*.module.ts`, `*.controller.ts`, `*.service.ts` | `NESTJS` |
| `oscode-web`, `oscode-admin`, `urbuddy-admin`, `urbuddy-web` | `app/`, `pages/`, `next.config.*` | `NEXTJS` |
| `oscode-rn` | `app.json`, `android/`, `ios/`, `*.native.tsx` | `EXPO_RN` |
| (unknown / mixed) | — | `GENERIC` |

Print detected project type: `Project: NESTJS | NEXTJS | EXPO_RN | GENERIC`. If a monorepo contains multiple packages, detect per changed file and apply checks accordingly.

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

---

### Project-Specific Checks (NestJS) — apply only when `PROJECT_TYPE = NESTJS`

- **Wrong date library** — direct import of `dayjs` or `moment` instead of `@oscode/dayjs`; flag any `new Date()` arithmetic that should use the shared package
- **String literal status comparison** — comparing status/state values with raw strings (`=== 'active'`) instead of Prisma-generated enums or shared enum constants; flag wherever a Prisma enum exists for the field
- **Wrong logger** — use of `console.*`, NestJS built-in `Logger`, or `winston` directly instead of `@oscode/nestjs-logger`; every `new Logger()` or `this.logger = new Logger()` call is a flag
- **PII in logs** — log statements that include user email, phone, password, token, national ID, or other personally identifiable fields — even in error paths
- **Common DTO not reused** — new DTO that duplicates a shape already exported from `@oscode/oscode-schema`; flag if a field set is ≥80% identical to a known shared DTO
- **Missing yaak file** — new controller endpoint added but no corresponding `.yaak` / yaak collection file created or updated; flag each new route that lacks a request entry
- **Flat module structure** — feature logic (service, controller, DTO, guard) placed directly in the module root instead of a sub-module folder; flag if the feature is non-trivial (>1 file)
- **Missing test files** — new service/controller with no matching `.spec.ts` (unit) and no `e2e` test file; flag each untested public method on a service
- **SOLID violations** — single service handling unrelated concerns (SRP); direct instantiation of dependencies instead of injection (DIP); service that cannot be swapped without changing callers (OCP)
- **DRY / KISS violations** — duplicated guard logic, repeated transformation chains that belong in a shared utility, or unnecessarily complex conditional flows

### Project-Specific Checks (Next.js) — apply only when `PROJECT_TYPE = NEXTJS`

- **Raw component instead of `@oscode/ui`** — use of `<button>`, `<input>`, `<select>`, `<dialog>`, etc. where `@oscode/ui` exports a component for it; flag by matching HTML element tags in `.tsx` files outside `@oscode/ui` itself
- **Broken component hierarchy** — component placed at the wrong layer: logic that belongs in an organism inside an atom, or a page-level concern embedded in a molecule; the rule is atoms → molecules → organs → organisms → pages
- **Hardcoded style value** — inline style (`style={{ color: '#fff' }}`), hardcoded Tailwind arbitrary value (`text-[#4f46e5]`), or a magic number not defined in `global.css` / `tailwind.config.*`
- **Design pattern / maintainability** — repeated prop-drilling more than two levels deep (use context or a shared store), duplicated component logic that belongs in a shared hook, or a component doing both data-fetching and rendering without separation

### Project-Specific Checks (Expo / React Native) — apply only when `PROJECT_TYPE = EXPO_RN`

- **Raw RN component instead of `@oscode/ui`** — use of `<View>`, `<Text>`, `<TouchableOpacity>`, `<TextInput>`, etc. where `@oscode/ui` exports a native equivalent; flag in non-`@oscode/ui` source files
- **Broken component hierarchy** — same rule as Next.js: atoms → molecules → organs → organisms; flag misplaced components
- **Hardcoded style value** — raw `StyleSheet.create` values, inline style numbers, or nativewind arbitrary values not anchored to `global.css` / `tailwind.config.*`
- **Nativewind / Tailwind config bypass** — color, spacing, or font value defined locally instead of in the global Tailwind theme; detect by comparing values against what's in `global.css`
- **Design pattern / maintainability** — same principles as Next.js; additionally flag platform-specific logic (`Platform.OS`) that should be abstracted behind a shared hook or utility

### Confidence gate

Only report findings you're ≥ 7/10 confident are real. Label 5–6/10 as `(TENTATIVE)`. Suppress anything below 5.

---

## Phase 3 — Specialist Dispatch

Skip all specialists for `--quick`.

Dispatch all five specialists **in parallel** using the Agent tool — one agent per specialist. Each agent receives the full diff and the detected `PROJECT_TYPE` as context and returns its findings. Collect results before writing the report.

### Security Specialist

Focus on the changed code only.

- **Hardcoded credentials** — API keys, tokens, passwords, private keys in diff
- **IDOR** — new endpoints that return or modify data keyed by a user-supplied ID without verifying ownership
- **CSRF** — state-mutating endpoints (POST/PUT/DELETE) missing CSRF protection where relevant
- **Injection paths** — user input flowing into shell commands (`exec`, `spawn`, `child_process`), template engines, or eval
- **Sensitive data in logs** — passwords, tokens, PII printed to stdout/logs/error messages (reinforces the NestJS PII check with a code-path trace)
- **Webhook signature bypass** — webhook receiver that doesn't verify the HMAC header (Stripe, GitHub, Slack patterns)

For each hit, trace the full code path: confirm user input actually reaches the sink before reporting.

### Testing Specialist

- **Missing happy-path test** — new function or endpoint with no corresponding test
- **Missing error-path test** — error branches (DB failure, auth failure, invalid input) with no test coverage
- **Test that doesn't assert** — test that calls code but only checks it doesn't throw, without asserting the actual output
- **Mocked-away reality** — test mocks the thing it's supposed to be testing (e.g., mocks the DB and then asserts the mock was called)
- **Snapshot drift** — snapshot file updated in the diff without a review comment explaining what changed visually; silent snapshot updates mask regressions
- **Regression gap** — changed behavior that existing tests no longer cover

**NestJS additions:** Confirm that new controllers have controller spec tests (`*.controller.spec.ts`), new services have service spec tests (`*.service.spec.ts`), and new endpoints have e2e tests (`*.e2e-spec.ts`). Flag each missing file.

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
- **Barrel import of large library** — `import * as Foo from 'large-lib'` or `import { everything } from 'lib'` where tree-shaking is blocked
- **New heavy dependency** — `package.json` change adds a dependency >50 KB gzipped with a lighter alternative available
- **Re-render trigger** — inline object/function created in render passed as a prop to a memoized component, defeating `React.memo`; `useEffect` dependency array includes an unstable reference
- **Missing `loading` / `Suspense` boundary** — new data-fetching component has no loading state, causing layout shift or blank content on slow connections

### API Contracts Specialist

- **Breaking response change** — field removed, renamed, or type-narrowed in a response that existing clients depend on
- **New required request field** — existing callers that don't send this field will now fail
- **Undocumented status code** — endpoint now returns a status code not in its spec/types
- **Pagination contract change** — cursor/offset scheme changed in a way that breaks clients mid-pagination
- **Auth requirement added** — previously public endpoint now requires auth (silent breaking change for integrations)
- **Missing yaak entry** — (NestJS) new route added but no yaak collection file created/updated

If the repo's `CLAUDE.md` routing table lists an owner for the changed path, note their handle next to findings in that area.

### Architecture & Design Patterns Specialist

Apply based on detected `PROJECT_TYPE`. Focus on maintainability, SOLID/KISS/DRY, and oscode conventions.

**All project types:**
- **DRY violation** — logic duplicated across two or more files that should live in a shared utility, hook, or service
- **KISS violation** — overly complex implementation for a straightforward requirement (deep nesting, unnecessary abstraction, multiple indirections for a single data transform)
- **Naming inconsistency** — entities, DTOs, or components named differently from established patterns in the same module/folder

**NestJS:**
- **SRP violation** — service class handling more than one distinct concern (e.g., email sending + user creation + audit logging all in one service)
- **DIP violation** — hard dependency on a concrete class instead of an injected interface/token
- **Missing sub-module** — feature-specific files (guard, interceptor, dto) placed at the module root when they belong in a dedicated sub-folder (e.g., `modules/user/auth/` not `modules/user/`)
- **Shared package bypass** — reimplementing something already provided by `@oscode/dayjs`, `@oscode/nestjs-logger`, or `@oscode/oscode-schema`

**Next.js / Expo RN:**
- **Component hierarchy violation** — atom importing from an organism, molecule with page-level side effects, organism duplicating molecule logic
- **UI package bypass** — rendering raw HTML/RN primitives when `@oscode/ui` provides the equivalent; flag the specific component and the `@oscode/ui` alternative
- **Tailwind config bypass** — styles defined outside `global.css` / `tailwind.config.*` that belong in the theme (colors, spacing, font sizes)
- **Prop drilling past two levels** — data passed through more than two component layers without a context or store; suggest the appropriate pattern

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
Project: NESTJS | NEXTJS | EXPO_RN | GENERIC
Plan: M/N items DONE  [or "No plan file found"]
Scope: ALIGNED  [or "SCOPE DRIFT — diff touches X but PR describes Y"]

#   Severity    Conf    Effort   Specialist          Finding                                File:Line
──  ─────────   ────    ──────   ──────────          ───────                                ─────────
1   BUG         9/10    Medium   Critical Pass       Null deref when session expires        src/auth.ts:42
2   SECURITY    8/10    Quick    Security            Token printed on failed login          src/api/login.ts:88
3   ARCH        8/10    Medium   Architecture        SRP: UserService handles email + auth  src/modules/user/user.service.ts:10
4   TESTS       7/10    Quick    Testing             No e2e test for POST /orders           src/modules/order/order.controller.ts:55
5   QUALITY     6/10    Hard     Performance (TENT.) Waterfall await in hot endpoint        src/api/search.ts:55

VERDICT: REQUEST CHANGES  (1 bug, 1 security, 1 arch issue)
```

**Effort key:** `Quick` = <15 min / a few lines; `Medium` = 15–60 min / meaningful refactor; `Hard` = >1 hour / design change needed.

### Finding format

```
## Finding N: [Title] — [file:line]

* **Severity:** BUG | SECURITY | QUALITY | TESTS | PERFORMANCE | ARCH
* **Confidence:** N/10
* **Effort:** Quick | Medium | Hard
* **Specialist:** Critical Pass | Security | Testing | Performance | API Contracts | Architecture
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
| ARCH | Design/structure violation that will cause maintainability pain; includes oscode convention violations |
| QUALITY | Will slow down future changes or mislead the next reader |

### Verdict

- **APPROVE** — no BUG or SECURITY findings; only QUALITY / TESTS / ARCH notes (or none)
- **APPROVE WITH SUGGESTIONS** — no blockers; QUALITY / TESTS / PERFORMANCE / ARCH findings worth addressing
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
- [ ] Project type detected and printed in report header
- [ ] Project-specific Phase 2 checks applied for the detected project type
- [ ] **NestJS:** `@oscode/dayjs`, `@oscode/nestjs-logger`, `@oscode/oscode-schema` usage checked
- [ ] **NestJS:** Enum usage for status comparisons verified; string literal comparisons flagged
- [ ] **NestJS:** PII fields absent from all log statements
- [ ] **NestJS:** Yaak file presence verified for every new controller route
- [ ] **NestJS:** Sub-module structure respected; flat placement of feature files flagged
- [ ] **NestJS:** Controller spec, service spec, and e2e test files checked for new endpoints
- [ ] **Next.js / Expo RN:** `@oscode/ui` usage verified; raw HTML/RN primitive substitutions flagged
- [ ] **Next.js / Expo RN:** Component hierarchy (atoms → molecules → organs → organisms) respected
- [ ] **Next.js / Expo RN:** No hardcoded style values outside `global.css` / `tailwind.config.*`
