---
name: benchmark
description: >-
  Measure page load performance using the browse daemon. Captures TTFB, FCP,
  LCP, bundle sizes, and request counts. Saves a baseline on first run;
  compares on subsequent runs and flags regressions. Invoke whenever the user
  asks about performance, page speed, load time, or bundle size.
---

# Benchmark

You are a performance engineer. Performance degrades through small increments — 50ms here, 20KB there. Your job is to measure exactly, compare against the last known-good state, and flag anything that crossed a threshold.

## When to invoke

User says: "benchmark", "page speed", "performance", "load time", "bundle size", "is this slower?", or invokes `/benchmark`.

## Arguments

- `/benchmark <url>` — full audit with baseline comparison
- `/benchmark <url> --baseline` — capture a fresh baseline (run before making changes)
- `/benchmark <url> --quick` — single timing pass, no baseline needed

## Workflow

### Step 1 — Verify browse is ready

```bash
B="$HOME/.claude/skills/gstack/browse/dist/browse"
[ -x "$B" ] && echo "READY" || echo "NEEDS_SETUP"
```

If `NEEDS_SETUP`: tell the user "browse needs a one-time build. Run `cd ~/.claude/skills/gstack && ./setup` then retry."  Stop here.

### Step 2 — Collect metrics

For each page being tested:

```bash
$B goto <url>
$B perf
$B eval "JSON.stringify(performance.getEntriesByType('navigation')[0])"
```

Extract from the navigation entry:
- **TTFB** = `responseStart - requestStart`
- **FCP** = first `paint` entry with name `first-contentful-paint`
- **DOM Interactive** = `domInteractive - navigationStart`
- **Full Load** = `loadEventEnd - navigationStart`

Collect resource summary:

```bash
$B eval "(() => {
  const r = performance.getEntriesByType('resource');
  return JSON.stringify({
    total_requests: r.length,
    total_transfer_kb: Math.round(r.reduce((s,e) => s + (e.transferSize||0), 0) / 1024),
    js_kb: Math.round(r.filter(e => e.initiatorType==='script').reduce((s,e) => s + (e.transferSize||0), 0) / 1024),
    css_kb: Math.round(r.filter(e => e.initiatorType==='stylesheet').reduce((s,e) => s + (e.transferSize||0), 0) / 1024),
    top5_slow: r.sort((a,b) => b.duration - a.duration).slice(0,5).map(e => ({
      name: e.name.split('/').pop().split('?')[0],
      type: e.initiatorType,
      size_kb: Math.round((e.transferSize||0)/1024),
      duration_ms: Math.round(e.duration)
    }))
  });
})()"
```

### Step 3 — Baseline

Baseline file lives at `.gstack/benchmark-baselines/<slug>.json` where slug is derived from the URL hostname + path (replace `/` with `-`, strip query strings). Create `.gstack/benchmark-baselines/` if it doesn't exist.

**`--baseline` flag or no existing baseline:** save current metrics as the baseline and report absolute numbers only. Tell the user: "Baseline saved. Run again after your changes to detect regressions."

**Baseline exists and no `--baseline` flag:** load it and compare (Step 4).

Baseline format:
```json
{
  "url": "<url>",
  "branch": "<git branch>",
  "captured_at": "<ISO timestamp>",
  "ttfb_ms": 0,
  "fcp_ms": 0,
  "dom_interactive_ms": 0,
  "full_load_ms": 0,
  "total_requests": 0,
  "total_transfer_kb": 0,
  "js_kb": 0,
  "css_kb": 0
}
```

### Step 4 — Compare and report

Print a table: baseline → current → delta → status.

```
PERFORMANCE REPORT — <url>
Branch: <current> vs baseline (<baseline-branch>, <baseline-date>)

Metric              Baseline    Current     Delta      Status
──────              ────────    ───────     ─────      ──────
TTFB                  120ms      135ms      +15ms      OK
FCP                   450ms      480ms      +30ms      OK
DOM Interactive       600ms      650ms      +50ms      OK
Full Load            1400ms     2100ms     +700ms      REGRESSION
Total Requests           42         58        +16      WARNING
Transfer Size         1.2MB      1.8MB      +0.6MB     REGRESSION
JS Bundle             450KB      720KB      +270KB     REGRESSION
CSS Bundle             85KB       88KB        +3KB     OK

REGRESSIONS: 3  WARNINGS: 1
```

**Thresholds:**

| Metric | WARNING | REGRESSION |
|--------|---------|------------|
| Timing (ms) | >20% increase | >50% increase OR >500ms absolute |
| Bundle size | >10% increase | >25% increase |
| Request count | >30% increase | >50% increase |

### Step 5 — Slow resources

Always print the top 5 slowest resources with type, size, and duration. Flag third-party domains with `[3p]`. One-line fix hint for anything over 200ms.

### Step 6 — Verdict

One of:
- **PASS** — no regressions, all warnings minor
- **WARN** — warnings present, no hard regressions
- **REGRESSION** — one or more regression thresholds crossed; list each with a concrete next step

## Rules

- Report what the browser measured. Never estimate or guess timings.
- Bundle size is the leading indicator — it's deterministic where load time varies with network.
- Flag third-party scripts but don't count them as regressions the user can fix.
- Read-only. Produce the report; do not modify code unless explicitly asked.
- If `--quick` is passed, skip baseline comparison and just report absolute numbers.
