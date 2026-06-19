---
name: sig-instrumentation-metrics-review
description: Review a kubernetes/kubernetes pull request that changes metrics/instrumentation code, applying SIG-Instrumentation approver standards (metric stability framework, naming conventions, cardinality, deprecation policy, component-base/metrics usage, and the verify-* tooling). Use when asked to review a k8s PR/diff that touches metrics, instrumentation, or the stable metrics list. Trigger phrases: "review this metrics PR", "sig-instrumentation review", "is this metric stable/named correctly".
---

# SIG-Instrumentation Metrics Review

Review a `kubernetes/kubernetes` pull request the way a **SIG-Instrumentation approver** would.
This skill reviews **only** metrics/instrumentation-relevant changes. If the PR doesn't touch
metrics, say so and stop.

## How these approvers actually review (verified patterns)

### 1. New metric → starts at ALPHA. Feature-gate stage ≠ metric maturity. (hard rule)
- **@rexagod** (#130951, changes requested): *"Feature graduations don't influence their corresponding
  metrics' maturity, which is governed by the metrics stability framework, as such, new metrics should
  kick off from `ALPHA`."*
- **@richabanker** (#130951): *"why is the new metric starting off at BETA stabilityLevel?"*
- **@rexagod** (#131768): *"While this FG is GA now, metrics should touch all stages when graduating,
  as such this should soak in BETA for atleast one release."* → promotions can't skip stages.

### 2. Tests must exercise the real code path, not call `Inc()`/`Observe()` directly. (most frequent)
- **@richabanker** (#136894): *"No, we dont want tests just invoking `Inc()` and verifying the metric
  is recorded, we rather woudl prefer invoking the underlying logic that should increment the metric
  instead to make sure the underlying code behavior is correct."* (cites
  `community/.../sig-instrumentation/metric-instrumentation.md`, "Adding a new metric" point #5).
- **@richabanker** (#137116): redundant metric tests should be dropped when *"`TestWatchEventSizes` …
  actually invokes the underlying logic responsible for incrementing these metrics, so those tests are
  more foolproof."*

### 3. Verify the *whole* metric with `testutil.GatherAndCompare`, not piecemeal asserts.
- **@richabanker** (#136154): *"Would prefer the `GatherAndCompare` instead of breaking down the checks
  in different `assert.Contains` calls."*
- **@richabanker** (#136189): *"is it possible to use the util `testutil.GatherAndCompare` to verify the
  whole metric struct…?"* — and for verbose histograms: *"we could … only compare the `_count` or
  `_sum` timeseries and ignore the individual bucket values."*
- **@serathius** (#135782): prefer registering into a fake registry and comparing golden `# HELP / #
  TYPE …` text over a hand-checked fake. **@richabanker** (#136189): don't stash expected values in
  separate files — *"Having new files for every metric test is probably not something we want."*

### 4. Pick the right instrument type.
- **@dashpole** (#136314): *"Why is this a gauge metric? It seems to be cumulative, and incremented like
  a counter, and count particular events."* → cumulative event counts = counter, not gauge.
- **@serathius** (#138767): *"Think that histogram would work better. The gauge will just show duration
  of last initialization as constant value… use a histogram"* → distributions = histogram, not gauge.

### 5. Histogram buckets must be bounded by a real-world limit and stay sensible across unit changes.
- **@serathius** (#138767): *"There is a timeout for watch initialization. Can you find the variable and
  use it as max bucket?"* → bound buckets by the actual operation limit.
- On unit changes (e.g. ms→s, ×1000), buckets must be re-scaled or values collapse into the first
  bucket and the histogram loses all resolution.

### 6. Naming/labels: consistent `Namespace`/`Subsystem` prefixes and help text.
- **@dgrisonnet** (#106737): *"set `Namespace: \"apiserver\"` and `Subsystem: \"watch_cache\"` in all
  the metric options … prefixed by `apiserver_watch_cache` which would make the naming … more
  consistent"*, plus *"some help texts are writing watchcache as `watchcache` and others as `watch
  cache`, could you make that consistent?"*
- Label design follows the group/version/resource convention (k8s#131796); **@serathius** (#135782):
  a duplicate metric identifier should *return an error*, not silently expose a metric under the wrong
  name (avoids confusing/duplicated series).

### 7. Deprecate, don't rename in place — even for ALPHA if it's been around.
- **@dashpole** (#136399): *"It is generally a good practice to mark the old one as deprecated."*
- **@dgrisonnet** (#106737): *"these metrics are ALPHA, we could theoretically rename them right away,
  but since some of them have been around for a while, it might be safer to have them go through a 2
  releases deprecation cycle and introduce new metrics as ALPHA."*

### 8. Metrics docs must stay in sync; a presubmit enforces it.
- **@pohly** (#138542, #139072) drove the convention for the verify script: *"write failures to stderr;
  include a `hack/update-metrics-documentation-list.sh` for symmetry and discoverability; mention
  running that script at the end of the failure message."* The list lives at
  `hack/tools/instrumentation/documentation/documentation-list.yaml`. Note (@serathius/@pohly debate on
  #139072): docs are updated per-PR now, but historically only at release — confirm current CI behavior.

### 9. Hygiene the approvers consistently enforce.
- **One concern per PR.** **@richabanker** (#136154): *"lets not merge flaky test fixes and metrics
  changes in the same PR?"* **@pohly** (#138384, changes requested): *"Can you squash into one commit?
  Your second one contains changes unrelated to it's commit message."*
- **Description/release notes must match reality.** **@richabanker** (#136154): *"PR description and
  release note seem inaccurate, I dont see the following metrics being promoted — `rest_client_requests_total`,
  `rest_client_request_duration_seconds`."*
- **Data races in hot-path instrumentation.** **@pohly** (#119949, #138567) repeatedly flags races
  introduced by metric/label construction (e.g. via `klog.KObj`).
- **New metrics package → add an `OWNERS` giving SIG-Instrumentation ownership** (**@serathius**,
  #139082), and place it to avoid odd cross-component imports (**@pohly**, #138542).
- **Copyright nit** (**@richabanker**, #136360/#136154): omit the year in new files' copyright headers.

## Step 1 — Get the change

Determine the target from the user's input:
- A PR number/URL → `gh pr view <n> --repo kubernetes/kubernetes` and `gh pr diff <n> --repo kubernetes/kubernetes`.
- A local checkout / current branch → `git diff` against the merge base.
- A pasted diff → use it directly.

If `gh` is unavailable or unauthenticated, fall back to the GitHub API
(`https://api.github.com/repos/kubernetes/kubernetes/pulls/<n>` and `.../files`) or ask the user to paste the diff.

## Step 2 — Scope to metrics

Confirm the change is in-scope. In-scope signals (any of):
- Touches `k8s.io/component-base/metrics` or imports it.
- Adds/changes a `metrics.NewCounterVec / NewGauge / NewHistogram / NewSummary` (the
  **component-base** wrappers, not raw `prometheus.NewX`).
- Edits `test/instrumentation/testdata/stable-metrics-list.yaml` (the generated stable-metrics snapshot)
  or `test/instrumentation/documentation/...`.
- Changes `StabilityLevel`, `DeprecatedVersion`, metric `Name`/`Subsystem`/`Namespace`, `Help`, `Buckets`, or labels.

If none apply, report "not a metrics change — out of SIG-Instrumentation scope" and stop.

## Step 3 — Review checklist

Go through each item. For every finding, cite the file:line and classify severity:
**BLOCKER** (must fix before approval), **NIT** (style/optional), **QUESTION** (need author clarification).

### A. Use the component-base wrappers, not raw Prometheus
- **BLOCKER** if new metrics are created with `prometheus.NewCounter/...` directly instead of
  `k8s.io/component-base/metrics`. The wrappers are what make the stability framework, hidden-metrics,
  and static analysis work. Raw client_golang metrics bypass all of it.
- Metrics must be registered through a `metrics.KubeRegistry` (e.g. `legacyregistry.MustRegister`),
  not `prometheus.MustRegister`.

### B. Naming conventions (`hack/verify-metrics-naming.sh`)
- snake_case, all lowercase.
- **Base units only**: `seconds` not `ms`/`milliseconds`, `bytes` not `kilobytes`. Append the unit.
- Counters MUST end in `_total`.
- Duration histograms/summaries SHOULD end in `_seconds` (e.g. `..._duration_seconds`).
- Don't encode label values into the metric name (no `requests_get_total` + `requests_post_total`;
  use a `verb` label instead — but watch cardinality, item C).
- Use a consistent `Namespace`/`Subsystem` prefix matching the component (e.g. `apiserver_`, `kubelet_`).

### C. Cardinality (the most common real-world problem)
- **BLOCKER/QUESTION** on unbounded label values: user IDs, full request paths, pod/namespace names,
  IPs, error strings, resource UIDs, full URLs. These explode series count and can OOM Prometheus.
- Prefer a bounded set. If a label must be bounded, require `metrics.ConstrainedLabels` /
  a label-value allowlist, or bucket the values.
- Sanity-check: `cardinality ≈ Π(distinct values per label)`. Flag anything that can grow with
  cluster size, request volume, or user input.

### D. Stability level (`StabilityLevel`)
- New metrics default to and should be `ALPHA` unless there's an explicit, justified promotion.
  **BLOCKER if a brand-new metric is introduced at `BETA`/`STABLE`** — a metric's maturity is governed
  by the stability framework, *not* by the maturity of the feature it measures. A GA feature gate does
  **not** justify a BETA/STABLE metric.
- **Promotions must walk every stage** (ALPHA→BETA→STABLE), soaking ≥1 release at BETA — they can't
  skip straight to STABLE just because the feature went GA.
- **Promotion to `STABLE`/`BETA` requires explicit SIG-Instrumentation sign-off** — flag it prominently
  and confirm there's a graduation rationale.
- Guarantees to enforce in review:
  - `ALPHA` / `INTERNAL`: no guarantees, may change/delete anytime.
  - `BETA`: labels may be **added** but **not removed**; ~1 release / 4 months min lifetime.
  - `STABLE`: name, type, labels, buckets are **frozen** — no labels added or removed; only change
    allowed is marking deprecated. **BLOCKER** if a STABLE metric's signature changes.

### E. Stable metrics list must be regenerated
- Any change affecting a STABLE/BETA metric must be reflected in
  `test/instrumentation/testdata/stable-metrics-list.yaml`.
- The author must run `hack/update-generated-stable-metrics.sh`; CI runs
  `hack/verify-generated-stable-metrics.sh`. **BLOCKER** if the metric changed but the snapshot didn't
  (or vice-versa) — the diff should be internally consistent.

### F. Deprecation / removal policy
- You cannot just delete or rename a STABLE/BETA metric. Lifecycle is
  **Stable → Deprecated (`DeprecatedVersion: "1.x"`) → Hidden → Deleted**.
- Minimum bake times before hiding/removal:
  - STABLE: ≥ 3 releases or 9 months after deprecation.
  - BETA: ≥ 1 release or 4 months.
  - ALPHA: same release is fine.
- Renames = deprecate old + add new (both for a transition period), not an in-place rename.
- Confirm `DeprecatedVersion` matches the actual target release.

### G. Instrument type (gauge vs counter vs histogram)
- A value that only ever **accumulates / counts events** must be a **counter** (`_total`), not a gauge
  (@dashpole, #136314).
- A **duration/size distribution** must be a **histogram**, not a gauge — a gauge only shows the last
  value as a flat line and can't give you rate or percentiles (@serathius, #138767).
- A point-in-time level (queue depth, in-flight) is a **gauge**. Flag mismatches as a BLOCKER.

### H. Histogram buckets
- Buckets should match the realistic value range and be powered/spaced sensibly (often
  `prometheus.ExponentialBuckets` / `DefBuckets`). Flag too-few buckets, buckets in the wrong unit,
  or buckets that don't cover the expected range.
- **Bound the top bucket by a real limit.** If the measured operation has a known timeout/cap, use that
  variable as the max bucket (@serathius, #138767) rather than an arbitrary number.
- **If a unit changed (e.g. ms→s), the buckets MUST be re-scaled** (ms→s is ×1000). Leaving buckets
  unchanged collapses values into the left-most bucket and destroys resolution — treat as a BLOCKER and
  ask "do these buckets still make sense at the new unit?"

### I. Documentation
- New/changed documented metrics must update the metrics documentation list, e.g.
  `hack/tools/instrumentation/documentation/documentation-list.yaml`, via
  `hack/update-metrics-documentation-list.sh` (CI: `hack/verify-metrics-documentation-list.sh`).
- `Help` text must be present, accurate, and describe units. Diff `Help`/doc wording against the prior
  text — reviewers catch spelling/regressions here.

### J. Tests (quality, not coverage theater)
- **BLOCKER: reject trivial metric tests.** A test that calls `Inc()`/`Observe()` directly and asserts
  the value moved tests the Prometheus library, not the code. The metric must be exercised by the
  behavior test for the surrounding feature (ref "Adding a new metric" point #5 in
  `metric-instrumentation.md`). If existing behavior tests already cover it, the new metric-only test
  is redundant — drop it.
- Use `k8s.io/component-base/metrics/testutil` (`GatherAndCompare` / `CollectAndCompare`) to assert the
  **whole** rendered metric (name, labels, `# HELP`, `# TYPE`) rather than many `assert.Contains` calls.
- For verbose histograms, compare only `_count`/`_sum` and ignore individual buckets.
- Keep expected golden text **inline in the test**, not in separate fixture files.

### K. PR hygiene the approvers enforce
- **One concern per PR**: don't mix unrelated fixes (e.g. flaky-test fix) with a metrics change; squash
  unrelated commits. Flag and ask to split.
- **Description & release notes must match the actual change** (e.g. list exactly which metrics are
  promoted). Flag mismatches.
- **New metrics package** must add an `OWNERS` assigning SIG-Instrumentation, and be placed to avoid odd
  cross-component imports.
- Watch for **data races** introduced in hot-path label/metric construction.

## Step 4 — Output

Produce a review in this format:

```
## SIG-Instrumentation Metrics Review — PR #<n>: <title>

**Scope:** <which metrics/files this touches>
**Verdict:** /lgtm | /approve | changes requested | needs SIG discussion

### Blockers
- [file:line] <issue> — <why it matters> → <fix>

### Nits
- [file:line] <suggestion>

### Questions for author
- <question>

### Required local commands before merge
- hack/verify-metrics-naming.sh
- hack/verify-generated-stable-metrics.sh   (run update-* first if it fails)
- hack/verify-metrics-documentation-list.sh
```

Be concrete, cite lines, and explain the *why* (stability guarantee, cardinality blast radius, prom
convention) — not just the rule. Default to `ALPHA` skepticism: question any promotion to STABLE/BETA.

## Reference
- `OWNERS_ALIASES` → `sig-instrumentation-approvers` / `-reviewers` define who can `/approve` and `/lgtm`.
- Metric stability framework & deprecation policy:
  `kubernetes/community` → `contributors/devel/sig-instrumentation/metric-stability.md`.
- Instrumentation guidance (incl. test guidance): `metric-instrumentation.md` in the same directory.
- Stability is implemented in `k8s.io/component-base/metrics`.

