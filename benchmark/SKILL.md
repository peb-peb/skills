---
name: benchmark
description: >-
  Measure page load performance using curl and shell tools. Captures TTFB, DNS,
  connect time, transfer sizes, and request counts. Saves a baseline on first
  run; compares on subsequent runs and flags regressions. Invoke whenever the
  user asks about performance, page speed, load time, or bundle size.
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

### Step 1 — Collect timing metrics

Use curl's built-in timing to measure primary network metrics:

```bash
curl -o /dev/null -s -w \
  'dns_ms:%{time_namelookup}\nconnect_ms:%{time_connect}\nssl_ms:%{time_appconnect}\nttfb_ms:%{time_starttransfer}\ntotal_ms:%{time_total}\nsize_bytes:%{size_download}\nhttp_code:%{http_code}\n' \
  "<url>"
```

All `time_*` values are in seconds — multiply by 1000 for ms. Extract:
- **DNS** = `time_namelookup * 1000`
- **Connect** = `(time_connect - time_namelookup) * 1000`
- **TTFB** = `time_starttransfer * 1000`
- **Full Load** = `time_total * 1000`
- **HTML size** = `size_download / 1024` KB

### Step 2 — Collect resource metrics

Fetch the HTML and extract resource URLs to measure JS/CSS bundle sizes:

```bash
# Fetch HTML
HTML=$(curl -s "<url>")

# Extract absolute and relative resource URLs (JS + CSS)
echo "$HTML" | grep -oE '(src|href)="[^"]*\.(js|css)[^"]*"' | sed 's/^[^"]*"//; s/"$//'
```

For each extracted resource URL (resolve relative paths against the base URL), measure transfer size:

```bash
curl -o /dev/null -s -w '%{size_download},%{time_total}\n' "<resource_url>"
```

Aggregate:
- **JS KB** = sum of `size_download` for `.js` resources / 1024
- **CSS KB** = sum of `size_download` for `.css` resources / 1024
- **Total Requests** = count of all resources found + 1 (the HTML itself)
- **Total Transfer KB** = (HTML size + JS KB + CSS KB + image sizes)
- **Top 5 Slow** = resources sorted by `time_total` descending, take first 5

If the page requires auth or returns a non-2xx for resources, note it and skip those resources.

### Step 3 — Baseline

Baseline file lives at `.benchmark-baselines/<slug>.json` where slug is derived from the URL hostname + path (replace `/` and `.` with `-`, strip query strings). Create `.benchmark-baselines/` if it doesn't exist.

**`--baseline` flag or no existing baseline:** save current metrics as the baseline and report absolute numbers only. Tell the user: "Baseline saved. Run again after your changes to detect regressions."

**Baseline exists and no `--baseline` flag:** load it and compare (Step 4).

Baseline format:
```json
{
  "url": "<url>",
  "branch": "<git branch>",
  "captured_at": "<ISO timestamp>",
  "ttfb_ms": 0,
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

Print the top 5 slowest resources with type, size, and duration. Flag third-party domains with `[3p]`. One-line fix hint for anything over 200ms.

### Step 6 — Verdict

One of:
- **PASS** — no regressions, all warnings minor
- **WARN** — warnings present, no hard regressions
- **REGRESSION** — one or more regression thresholds crossed; list each with a concrete next step

## Rules

- Report what curl measured. Never estimate or guess timings.
- Bundle size is the leading indicator — it's deterministic where load time varies with network.
- Flag third-party scripts but don't count them as regressions the user can fix.
- Read-only. Produce the report; do not modify code unless explicitly asked.
- If `--quick` is passed, skip resource scraping and baseline comparison — report only curl timing and HTML size.
- These are wire-level metrics (no JS execution, no render). Note this in the report header.
