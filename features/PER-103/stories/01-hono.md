# Story PER-103 — Fix TrieRouter 404 when regex parameter is followed by literal segment

**Type:** bug  
**ID:** 103  
**Parent:** (none)  
**Status:** planned  
**Assignee:** (unassigned)  
**Priority:** high  
**Severity:** 2-major  

---

## Description

`TrieRouter` returns `404` for `GET /a/42/[draft]` when the route is registered as `/a/:id{[0-9]+}/[draft]`. The segment `[draft]` is a literal path segment, but `getPattern()` in `src/utils/url.ts` incorrectly treats the brackets as a regex character class in the lookahead pattern. `RegExpRouter` and `LinearRouter` handle this correctly and return `200` with body `42`. The default app uses `SmartRouter`, which selects `RegExpRouter` for this single route but switches to `TrieRouter` when unrelated routes are added, causing the same request to fail.

**Confirmed behavior:**

```
TrieRouter                         404 Not Found
RegExpRouter                       200 42
LinearRouter                       200 42
SmartRouter (single route)         200 42 (uses RegExpRouter)
SmartRouter (with extra routes)    404 Not Found (uses TrieRouter)
```

**Root cause:** `getPattern()` builds a lookahead for the next segment without escaping bracket literals when the current segment is a regex parameter. `LinearRouter` (`src/router/linear-router/router.ts`) constructs this lookahead separately and does not exhibit the bug.

---

## Technical Specification

### Problem Analysis

When a route pattern contains a regex parameter followed by a literal segment with brackets (e.g., `/a/:id{[0-9]+}/[draft]`), `getPattern()` in `src/utils/url.ts` generates a lookahead pattern that misinterprets `[draft]` as a character class `[draf]` (matching 'd', 'r', 'a', or 'f') instead of the literal string `[draft]`.

**Current behavior in `getPattern()`:**
- For a regex parameter like `:id{[0-9]+}`, the function builds a lookahead to match the next segment
- The lookahead is constructed by concatenating the next segment's pattern directly into the regex
- When the next segment is a literal like `[draft]`, the brackets are not escaped
- Result: `/a/:id{[0-9]+}/[draft]` becomes a pattern where `[draft]` is interpreted as `[draf]`

**Correct behavior (as in `LinearRouter`):**
- `LinearRouter` in `src/router/linear-router/router.ts` builds lookahead patterns that escape bracket literals
- The lookahead for `[draft]` becomes `\[draft\]`, which matches the literal string `[draft]`

### Solution

**File to change:** `src/utils/url.ts`

**Function to fix:** `getPattern()`

**Change required:**
When building the lookahead pattern for the next segment after a regex parameter, escape `[` and `]` so that literal brackets in segment names are not interpreted as character classes.

**Implementation notes:**
- Locate the code in `getPattern()` that constructs the lookahead for the next segment when processing a regex parameter
- Before inserting the next segment's pattern into the lookahead, escape any literal `[` and `]` characters
- The escaping must apply only to literal segment names, not to segments that are themselves regex parameters (those brackets are intentional)
- Verify that segments without brackets (e.g., `file.html`) continue to work as before

**Files affected:**
- `src/utils/url.ts` — fix in `getPattern()`
- `src/router/trie-router/router.test.ts` — add regression tests for TrieRouter
- `src/router/smart-router/router.test.ts` — add tests for SmartRouter fallback behavior
- `src/router/reg-exp-router/router.test.ts` — verify existing behavior (no changes needed)
- `src/router/linear-router/router.test.ts` — verify existing behavior (no changes needed)

**Constraints:**
- Do not change route syntax or the public `Hono` constructor
- Do not modify `RegExpRouter` or `LinearRouter` behavior
- Open the pull request on `pavithrachandran70/hono`, not `honojs/hono`

---

## Acceptance Criteria

### AC1: TrieRouter matches the literal segment route

**Given** a route registered as `/a/:id{[0-9]+}/[draft]`  
**When** `GET /a/42/[draft]` is requested using `TrieRouter`  
**Then** the response is `200` with body `42`

**Verify:** Add test case to `src/router/trie-router/router.test.ts` that registers the route on a `TrieRouter` instance, makes the request, and asserts `res.status === 200` and `await res.text() === '42'`.

---

### AC2: RegExpRouter matches the literal segment route

**Given** a route registered as `/a/:id{[0-9]+}/[draft]`  
**When** `GET /a/42/[draft]` is requested using `RegExpRouter`  
**Then** the response is `200` with body `42`

**Verify:** Run existing test suite in `src/router/reg-exp-router/router.test.ts`. No new test needed; verify no regressions.

---

### AC3: LinearRouter matches the literal segment route

**Given** a route registered as `/a/:id{[0-9]+}/[draft]`  
**When** `GET /a/42/[draft]` is requested using `LinearRouter`  
**Then** the response is `200` with body `42`

**Verify:** Run existing test suite in `src/router/linear-router/router.test.ts`. No new test needed; verify no regressions.

---

### AC4: SmartRouter matches the literal segment route (single route)

**Given** the default app uses `SmartRouter` and has only the route `/a/:id{[0-9]+}/[draft]` registered  
**When** `GET /a/42/[draft]` is requested  
**Then** `SmartRouter` selects `RegExpRouter` and the response is `200` with body `42`

**Verify:** Add test case to `src/router/smart-router/router.test.ts` that registers only the route `/a/:id{[0-9]+}/[draft]` on a default `Hono` app (which uses `SmartRouter`), makes the request, and asserts `res.status === 200` and `await res.text() === '42'`.

---

### AC5: SmartRouter matches the literal segment route after fallback to TrieRouter

**Given** the default app uses `SmartRouter` and has `/a/:id{[0-9]+}/[draft]` plus unrelated routes registered  
**When** `GET /a/42/[draft]` is requested  
**Then** `SmartRouter` selects `TrieRouter` and the response is `200` with body `42`

**Verify:** Add test case to `src/router/smart-router/router.test.ts` that registers `/a/:id{[0-9]+}/[draft]` and several unrelated routes (enough to trigger `SmartRouter`'s fallback to `TrieRouter`), makes the request, and asserts `res.status === 200` and `await res.text() === '42'`.

---

### AC6: Non-matching paths remain unmatched

**Given** a route registered as `/a/:id{[0-9]+}/[draft]`  
**When** `GET /a/42/draft` (no brackets) is requested  
**Then** the route does not match  
**And when** `GET /a/42/[other]` is requested  
**Then** the route does not match

**Verify:** Add test cases to `src/router/trie-router/router.test.ts` that register the route and assert `res.status === 404` for both `/a/42/draft` and `/a/42/[other]`.

---

### AC7: Existing static segment routes still match

**Given** an existing route whose next segment after a regex parameter is `file.html` (or another literal without brackets)  
**When** a request matching that route is made  
**Then** the route matches as before the fix

**Verify:** Add test case to `src/router/trie-router/router.test.ts` that registers `/a/:id{[0-9]+}/file.html`, makes the request `GET /a/42/file.html`, and asserts `res.status === 200`.

---

### AC8: Regression tests cover TrieRouter and SmartRouter fallback

**Given** the fix is implemented  
**When** new tests are added to `src/router/trie-router/router.test.ts`  
**Then** the tests cover AC1, AC6, and AC7 for `TrieRouter`  
**And when** tests are added to `src/router/smart-router/router.test.ts`  
**Then** the tests cover AC4 and AC5 for `SmartRouter`

**Verify:** Review test files and confirm test cases exist that match the scenarios in AC1, AC4, AC5, AC6, and AC7.

---

### AC9: Existing router test suites pass

**Given** the fix is implemented  
**When** the test suites for `RegExpRouter` and `LinearRouter` are run  
**Then** all tests pass without modification (verifying AC2, AC3, and no regressions)

**Verify:** Run `pnpm test src/router/reg-exp-router/router.test.ts` and `pnpm test src/router/linear-router/router.test.ts` and confirm all tests pass.

---

### AC10: Test and lint pass

**Given** the fix and tests are implemented  
**When** `pnpm test` is run  
**Then** all tests pass  
**And when** `pnpm lint` is run  
**Then** no lint errors are reported

**Verify:** Run `pnpm test` and `pnpm lint` in the repo root and confirm both exit with code 0.

---

## Notes

- The fix is localized to `src/utils/url.ts` in the `getPattern()` function.
- `LinearRouter` in `src/router/linear-router/router.ts` already handles this correctly and serves as a reference for the correct lookahead construction.
- The route syntax and public API remain unchanged.
- Pull request target: `pavithrachandran70/hono` (not `honojs/hono`).

---

## References

- Ticket: PER-103
- Plan: `story-per-103-plan.md`
- Requirements: `requirements.md`
- Repo: `hono` (`pavithrachandran70/hono`)
- Base branch: `main`

---

## Open Questions

None.
