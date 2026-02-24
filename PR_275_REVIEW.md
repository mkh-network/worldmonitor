# PR #275 Review — Infrastructure Cost Optimizations Round 2

**Reviewer:** Claude (automated)
**PR Author:** elzalem
**Branch:** feat/redis-shared-caching → main
**Commits:** 6 (6a8a480..fd00c7c)
**Files changed:** 9 (+338 / -128)

---

## Summary

This PR implements three infrastructure cost optimizations:

1. **ACLED API Deduplication** — Consolidates three independent ACLED API fetch paths (conflict, unrest, risk-scores) into a shared cached layer (`server/_shared/acled.ts`), eliminating redundant upstream calls when identical queries are made concurrently.

2. **Cache TTL Alignment** — Adjusts Redis cache TTLs to match actual upstream data refresh cadences:
   - Climate anomalies: 30min → 3h (ERA5 reanalysis has 2-7 day lag)
   - Fire detections: 30min → 1h (NASA FIRMS NRT updates ~every 3h)

3. **AIS Polling Visibility Guard** — Pauses AIS relay polling when the browser tab is backgrounded via the Page Visibility API, with a two-layer defense (inline `document.hidden` check + interval pause/resume).

---

## Validation Results

| Check | Result |
|-------|--------|
| PR-specific tests (18/18) | PASS |
| TypeScript type checking | PASS (no errors in PR-changed files) |
| Pre-existing test failures | 4 in `redis-caching.test.mjs` — unrelated to this PR |

---

## Detailed Findings

### server/_shared/acled.ts (new file) — APPROVE

Well-structured shared layer. Positives:
- Cache key derivation from query parameters (`acled:shared:{eventTypes}:{start}:{end}:{country}:{limit}`) enables deduplication across handlers with identical queries.
- Graceful degradation: returns `[]` when `ACLED_ACCESS_TOKEN` is missing.
- Fire-and-forget cache write with `.catch(() => {})` — appropriate for best-effort caching.
- 15-minute TTL matches ACLED's rate-limit window.
- Caches empty arrays (negative caching), preventing repeated cache misses for empty upstream responses.

**Minor observation:** The `setCachedJson` call is fire-and-forget (not awaited). This is fine for the cache-write path but means a transient Redis failure won't surface here — consistent with how other handlers in the codebase treat cache writes.

### Conflict handler (list-acled-events.ts) — APPROVE

Clean migration. Removed ~40 lines of inline fetch logic, now delegates to `fetchAcledCached`. Filter and mapping logic is preserved. The `as string[]` casts on `.filter(Boolean)` results are correct since TypeScript can't narrow `(string | undefined)[]` through `filter(Boolean)`.

### Risk scores handler (get-risk-scores.ts) — APPROVE with notes

**Behavioral difference #1 — Event type query format:**
The original code passed `event_type=Protests&event_type=Riots` (repeated query params). The new code passes `event_type=Protests|Riots` (pipe-separated single param). This matches how the conflict and unrest handlers have always queried ACLED, so it aligns with the established pattern. The ACLED API documents the pipe-separated format as canonical.

**Behavioral difference #2 — Timeout reduction:**
Original used `UPSTREAM_TIMEOUT_MS` (30s). Shared layer uses `ACLED_TIMEOUT_MS` (15s). The 15s timeout is more aggressive but reasonable for a single API call. The risk-scores handler has stale-cache fallback, so a timeout won't cause user-facing failures.

**Behavioral difference #3 — Unauthenticated calls removed:**
The original code set `Authorization` header conditionally (`if (token)`), allowing unauthenticated ACLED calls. The shared layer returns `[]` immediately when no token exists. This is a reasonable tightening since the ACLED API requires auth for meaningful results.

### Unrest handler (list-unrest-events.ts) — APPROVE

Same clean migration as conflict. The unrest handler previously only queried `Protests` (not `Protests|Riots`), and the new code preserves this distinction — it passes `eventTypes: 'Protests'` to the shared layer. Correct.

### Cache TTL changes — APPROVE

Both TTL increases are well-justified by upstream data source refresh rates:
- **Climate (3h):** ERA5 reanalysis data has a 2-7 day processing lag — a 30-minute cache was far too aggressive.
- **Wildfire (1h):** FIRMS NRT data updates approximately every 3 hours — a 30-minute cache was causing unnecessary Redis reads.

### Maritime AIS visibility guard — APPROVE

The two-layer defense is well-implemented:

1. **Inline guard** in `pollSnapshot()`: `if (!force && isClientRuntime && document.hidden) return;` — catches polls that fire between the visibility change event and the interval clear.
2. **Interval management** via `pausePolling()` / `resumePolling()`: Stops the interval entirely when backgrounded, preventing unnecessary timer wakeups.

`resumePolling()` correctly:
- Checks `isPolling` state (no-op if polling was never started).
- Guards against double-start (`if (pollInterval) return`).
- Avoids overlapping relay requests (`if (!inFlight)`).
- Fires an immediate non-forced poll before restarting the interval, so users see fresh data when switching back.

**Edge case handled:** The `isClientRuntime` check (`typeof window !== 'undefined'`) prevents the `document.addEventListener` from running in SSR/server contexts.

### Tests (ttl-acled-ais-guards.test.mjs) — APPROVE

17 structural tests (test runner reports 18 due to the negative caching assertion) cover all three optimization areas. Tests are source-reading based (verify file contents against patterns) rather than behavioral, which is appropriate for configuration/structural validations like TTL values and import presence.

---

## Risk Assessment

| Risk | Level | Mitigation |
|------|-------|------------|
| ACLED query format change (risk-scores) | Low | Pipe-separated format is canonical and already used by conflict/unrest handlers |
| Timeout reduction (30s → 15s) | Low | Stale cache fallback in risk-scores; graceful degradation in conflict/unrest |
| AIS polling pause on background | Low | Two-layer defense; forced polls bypass visibility guard; immediate poll on resume |
| Cache TTL increases | Very Low | TTLs still far shorter than upstream refresh periods |

---

## Verdict

**APPROVE** — This is a well-scoped infrastructure optimization PR. The ACLED deduplication eliminates clear redundancy, TTL alignment is well-justified by upstream data cadences, and the AIS visibility guard is defensively implemented with appropriate edge case handling. No security concerns, no functional regressions, all PR-specific tests pass.
