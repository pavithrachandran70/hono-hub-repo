# BMAD consistency check — PER-105

**First line:** FAIL

---

## Concrete mismatches

### 1. AC count mismatch (requirements vs breakdown vs stories)

**requirements.md** declares **7 acceptance criteria** (AC-1 through AC-7).

**breakdown.md** assigns ACs to stories as follows:
- Story 1 (PER-106): AC-5, AC-6, AC-7 → **3 ACs**
- Story 2 (PER-107): AC-1, AC-2 → **2 ACs**
- Story 3 (PER-108): AC-3, AC-4 → **2 ACs**
- Story 4 (PER-109): AC-7 → **1 AC** (duplicated from Story 1)
- Story 5 (PER-110): AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7 → **7 ACs** (all consolidated)

**Story 1 LLD (stories/01-hono.md)** lists AC-5, AC-6, AC-7 in its Acceptance Criteria section → **3 ACs** (matches breakdown Story 1).

**Story 2 LLD (stories/02-hono.md)** lists AC-1, AC-2 in its Acceptance Criteria section → **2 ACs** (matches breakdown Story 2).

**Story 3 LLD (stories/03-hono.md)** lists AC-3, AC-4 in its Acceptance Criteria section → **2 ACs** (matches breakdown Story 3).

**Story 4 LLD (stories/04-hono.md)** lists **only AC-7** in its Acceptance Criteria section → **1 AC** (matches breakdown Story 4, but AC-7 is duplicated from Story 1).

**Story 5 LLD (stories/05-hono.md)** lists AC-1 through AC-7 in its Acceptance Criteria section → **7 ACs** (matches breakdown Story 5).

**Mismatch:** AC-7 is assigned to both Story 1 and Story 4 in breakdown.md and both story LLDs. The breakdown declares Story 1 covers AC-7 via implementation ("assignment logic unchanged"), and Story 4 covers AC-7 via manual verification ("verify existing behavior unchanged"). This is a **duplication** rather than a count error, but it creates ambiguity: which story owns AC-7?

**Clarification needed:** AC-7 (header merging preserved) is verified in two places:
- Story 1 implements the cleanup mechanism and preserves the existing `this._res = res` assignment logic (implicit coverage).
- Story 4 manually verifies that header merging still works in other middleware (explicit coverage).

This is acceptable as long as both stories confirm AC-7 is met. However, the breakdown should clarify that AC-7 is **verified** in Story 4, not **implemented** in Story 4, since the implementation is in Story 1.

---

### 2. Test count mismatch (breakdown vs story LLDs)

**breakdown.md Summary table** declares:
- Story 1: **4 tests**
- Story 2: **4 tests**
- Story 3: **4 tests**
- Story 4: **0 tests** (manual verification only)
- Story 5: **12 tests**
- **Total:** 24 tests

**Story 1 LLD (stories/01-hono.md)** declares **4 tests** in the Tests section (Test 1, Test 2, Test 3, Test 4) → matches breakdown ✓

**Story 2 LLD (stories/02-hono.md)** declares **4 tests** in the Tests section (Test 1, Test 2, Test 3, Test 4) → matches breakdown ✓

**Story 3 LLD (stories/03-hono.md)** declares **4 tests** in the Tests section (Test 1, Test 2, Test 3, Test 4) → matches breakdown ✓

**Story 4 LLD (stories/04-hono.md)** declares **0 tests** (manual verification only) → matches breakdown ✓

**Story 5 LLD (stories/05-hono.md)** declares **12 tests** in the Tests section (Test 1 through Test 12) → matches breakdown ✓

**Total test count:** 4 + 4 + 4 + 0 + 12 = **24 tests** → matches breakdown ✓

**No mismatch.** Test counts are consistent.

---

### 3. File path mismatch (HLD vs breakdown vs story LLDs)

**HLD (hld.md)** references the following file paths:
- `src/context.ts` — Context response replacement (Component Design section)
- `src/middleware/etag/index.ts` — ETag middleware (assumed path, marked "verify in repo tree")
- `src/middleware/pretty-json/index.ts` — PrettyJSON middleware (assumed path, marked "verify in repo tree")
- Other middleware paths assumed: `src/middleware/cache/`, `src/middleware/compress/`, etc. (marked "verify paths in repo tree")
- `src/context.test.ts` — Unit tests (Context cleanup mechanism, Testing Strategy section)
- `src/middleware/etag/index.test.ts` — Regression tests (ETag middleware, marked "or new test file")
- `src/middleware/pretty-json/index.test.ts` — Regression tests (PrettyJSON middleware, marked "or new test file")

**breakdown.md** declares the following allowed paths:
- Story 1: `src/context.ts`, `src/context.test.ts`
- Story 2: `src/middleware/etag/index.test.ts`, `src/middleware/etag/index.ts` (no changes)
- Story 3: `src/middleware/pretty-json/index.test.ts`, `src/middleware/pretty-json/index.ts` (no changes)
- Story 4: `src/middleware/cache/`, `src/middleware/compress/`, `src/middleware/timeout/`, `src/middleware/jsx-renderer/`, `src/middleware/secure-headers/` (no changes, verification only)
- Story 5: `src/context.test.ts` (or `src/context.cleanup.test.ts` if splitting)

**Story 1 LLD (stories/01-hono.md)** references:
- `src/context.ts` — Core cleanup implementation ✓
- `src/context.test.ts` — Unit tests for cleanup behavior ✓

**Story 2 LLD (stories/02-hono.md)** references:
- `src/middleware/etag/index.test.ts` — Regression tests (marked "will be created" if it does not exist)
- `src/middleware/etag/index.ts` — No changes (cleanup is automatic)

**Story 3 LLD (stories/03-hono.md)** references:
- `src/middleware/pretty-json/index.test.ts` — Regression tests (marked "may or may not exist per repo-tree")
- `src/middleware/pretty-json/index.ts` — No changes (cleanup is automatic)

**Story 4 LLD (stories/04-hono.md)** references:
- `src/middleware/cache/` — Verification only (no changes)
- `src/middleware/compress/` — Verification only (no changes)
- `src/middleware/timeout/` — Verification only (no changes)
- `src/middleware/jsx-renderer/` — Verification only (no changes)
- `src/middleware/secure-headers/` — Verification only (no changes)

**Story 5 LLD (stories/05-hono.md)** references:
- `src/context.test.ts` — Comprehensive test suite (or `src/context.cleanup.test.ts` if splitting)

**repo-tree/hono verification:**
- `src/context.ts` → **exists** ✓
- `src/context.test.ts` → **exists** ✓
- `src/middleware/etag/index.test.ts` → **does not exist** (will be created in Story 2)
- `src/middleware/etag/index.ts` → **does not appear in repo-tree** (assumed to exist, but not listed — this is a **mismatch**)
- `src/middleware/pretty-json/index.test.ts` → **does not exist** (will be created in Story 3)
- `src/middleware/pretty-json/index.ts` → **does not appear in repo-tree** (assumed to exist, but not listed — this is a **mismatch**)
- `src/middleware/cache/` → **does not appear in repo-tree** (assumed to exist, but not listed — this is a **mismatch**)
- `src/middleware/compress/` → **does not appear in repo-tree** (assumed to exist, but not listed — this is a **mismatch**)
- `src/middleware/timeout/` → **does not appear in repo-tree** (assumed to exist, but not listed — this is a **mismatch**)
- `src/middleware/jsx-renderer/` → **does not appear in repo-tree** (assumed to exist, but not listed — this is a **mismatch**)
- `src/middleware/secure-headers/` → **does not appear in repo-tree** (assumed to exist, but not listed — this is a **mismatch**)

**Mismatch:** The HLD, breakdown, and story LLDs all reference middleware paths (`src/middleware/etag/`, `src/middleware/pretty-json/`, `src/middleware/cache/`, etc.) that are **not listed in the repo-tree**. The repo-tree cuts off at `src/helper/cookie/index.ts`, so the `src/middleware/` directory is **not shown** in the provided repo-tree. This is likely a **repo-tree truncation issue** rather than a file path invention, but it cannot be confirmed without the full repo-tree.

**Resolution needed:** The planner must verify that the `src/middleware/` directory exists in the actual repository and that the assumed paths (`etag/index.ts`, `pretty-json/index.ts`, etc.) are correct. If the repo-tree is truncated, request the full tree. If the middleware paths are wrong, correct them in the HLD, breakdown, and story LLDs.

---

### 4. HLD outcome vs LLD implementation mismatch

**HLD (hld.md) Component Design — Context response replacement** declares:
- The cleanup mechanism is internal to the `Context` class.
- The `res` setter (or a new internal `setRes()` method) is extended to invoke cleanup before assigning the new response.
- The implementation flow is:
  1. Check if `this._res` exists (previous response is set).
  2. Check if `this._res.body !== newResponse.body` (not reusing the same body).
  3. If both conditions are true, invoke `this._res.body?.cancel()` in a `try/catch` block.
  4. On error, log in development or ignore in production.
  5. Assign `newResponse` to `this._res`.
  6. Apply existing header merging logic (unchanged).

**Story 1 LLD (stories/01-hono.md) src/context.ts — Implementation sketch:**

```typescript
set res(res: Response) {
  // Cleanup previous body if replacing with a different body
  if (this._res && this._res.body !== res.body && this._res.body) {
    try {
      this._res.body.cancel()
    } catch (e) {
      // Development: log error for debugging
      // Production: silently ignore
      if (process.env.NODE_ENV === 'development') {
        console.error('Failed to cancel previous response body:', e)
      }
    }
  }
  this._res = res
}
```

**Comparison:**
- HLD step 1: "Check if `this._res` exists" → LLD: `if (this._res ...)`  ✓
- HLD step 2: "Check if `this._res.body !== newResponse.body`" → LLD: `this._res.body !== res.body` ✓
- HLD step 3: "Invoke `this._res.body?.cancel()` in a try/catch block" → LLD: `this._res.body.cancel()` (no optional chaining `?.`) ⚠️
- HLD step 4: "On error, log in development or ignore in production" → LLD: `if (process.env.NODE_ENV === 'development') { console.error(...) }` ✓
- HLD step 5: "Assign `newResponse` to `this._res`" → LLD: `this._res = res` ✓
- HLD step 6: "Apply existing header merging logic (unchanged)" → LLD: (not shown, but implied by unchanged assignment) ✓

**Mismatch (minor):** The HLD suggests optional chaining (`this._res.body?.cancel()`), but the LLD implementation uses `this._res.body.cancel()` without optional chaining. However, the LLD adds an explicit check `&& this._res.body` before the try/catch block, which achieves the same result (null/undefined check). This is **consistent in outcome** but **inconsistent in syntax**. The HLD should be clarified to match the LLD's explicit check pattern, or the LLD should use optional chaining as the HLD suggests.

**Recommended fix:** Update the HLD to reflect the LLD's explicit check pattern (preferred, as it is more explicit), or update the LLD to use optional chaining as the HLD suggests (acceptable, as it is more concise).

---

### 5. Missing `specs/<id>.md` files for child tickets

**breakdown.md** declares 5 child tickets:
- Story 1: `01-hono` → `PER-106`
- Story 2: `02-hono` → `PER-107`
- Story 3: `03-hono` → `PER-108`
- Story 4: `04-hono` → `PER-109`
- Story 5: `05-hono` → `PER-110`

**tickets.md** confirms the same mapping.

**specs/ directory** contains:
- `specs/1.md` → Covers all 5 child tickets (PER-106 through PER-110) in one epic-level spec

**Mismatch:** The epic spec (`specs/1.md`) covers the entire feature, not individual child tickets. The breakdown structure (5 stories, 5 tickets) suggests that each child ticket should have its own `specs/<id>.md` file, but only one epic spec exists. This is **acceptable** if the planner intended an epic-level spec, but **inconsistent** with the breakdown's per-story structure.

**Clarification needed:** The breakdown treats each story as a separate ticket (PER-106, PER-107, etc.), but the spec treats the entire feature as one epic. Either:
- The breakdown should be restructured to have one parent epic ticket (PER-105) with 5 child stories (not separate tickets), and `specs/1.md` is the epic spec (current state).
- Or, the spec should be split into 5 separate files (`specs/1.md` for PER-106, `specs/2.md` for PER-107, etc.), each covering its respective story.

**Current state matches:** The epic-level spec in `specs/1.md` is consistent with the proposed_tickets.md (1 ticket covering AC-1 through AC-7), but **not** with breakdown.md (5 separate stories/tickets). This is a **structural inconsistency**.

---

### 6. `merged_plan.md` missing

**plan.md Step 6** declares:
- `merge_pack (pack)` _(auto)_ — Pack per-ticket specs into `merged_plan.md` for review.

**Artifact list** does not include `merged_plan.md`.

**Mismatch:** The plan declares that `merged_plan.md` should be created by the `merge_pack` step, but it is **missing** from the provided artifacts. This suggests the `merge_pack` step has **not run yet**, or the artifact was not included in the consistency check input.

**Expected:** If `merge_pack` has run, `merged_plan.md` should exist and consolidate the per-ticket specs. If it has not run, the plan should reflect this (Step 6 is not yet complete).

---

### 7. `change_requests.md` not present (no contradiction check needed)

**Task:** "If change_requests.md is present, FAIL when artifacts still contradict it."

**Artifact list:** No `change_requests.
