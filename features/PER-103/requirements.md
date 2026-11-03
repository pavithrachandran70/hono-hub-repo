# Requirements — PER-103

**Story:** Fix TrieRouter 404 when regex parameter is followed by literal segment

## Description

`TrieRouter` returns `404` for `GET /a/42/[draft]` when the route is registered as `/a/:id{[0-9]+}/[draft]`. The segment `[draft]` is a literal path segment, but `getPattern()` in `src/utils/url.ts` incorrectly treats the brackets as a regex character class in the lookahead pattern. `RegExpRouter` and `LinearRouter` handle this correctly and return `200` with body `42`. The default app uses `SmartRouter`, which selects `RegExpRouter` for this single route but switches to `TrieRouter` when unrelated routes are added, causing the same request to fail.

**Root cause:** `getPattern()` builds a lookahead for the next segment without escaping bracket literals when the current segment is a regex parameter. `LinearRouter` (`src/router/linear-router/router.ts`) constructs this lookahead separately and does not exhibit the bug.

**Technical notes:**

- The fix is in `src/utils/url.ts` — `getPattern()` must escape `[` and `]` in the next segment when building the lookahead for a regex parameter, so `[draft]` becomes `\[draft\]` in the pattern.
- `LinearRouter` in `src/router/linear-router/router.ts` already handles this correctly by building the lookahead pattern in a way that does not misinterpret bracket literals.
- A following static segment such as `file.html` (no brackets) already works and must continue to work.
- Do not change route syntax, the public `Hono` constructor, or the behavior of `RegExpRouter` or `LinearRouter`.
- Open the pull request on `pavithrachandran70/hono`, not `honojs/hono`.

**Files affected:**

- `src/utils/url.ts` — fix in `getPattern()`
- `src/router/trie-router/router.test.ts` — add regression tests
- `src/router/smart-router/router.test.ts` — verify fallback behavior
- `src/router/reg-exp-router/router.test.ts` — verify existing behavior still passes
- `src/router/linear-router/router.test.ts` — verify existing behavior still passes

## Acceptance Criteria

### AC1: TrieRouter matches the literal segment route

**Given** a route registered as `/a/:id{[0-9]+}/[draft]`  
**When** `GET /a/42/[draft]` is requested using `TrieRouter`  
**Then** the response is `200` with body `42`

### AC2: RegExpRouter matches the literal segment route

**Given** a route registered as `/a/:id{[0-9]+}/[draft]`  
**When** `GET /a/42/[draft]` is requested using `RegExpRouter`  
**Then** the response is `200` with body `42`

### AC3: LinearRouter matches the literal segment route

**Given** a route registered as `/a/:id{[0-9]+}/[draft]`  
**When** `GET /a/42/[draft]` is requested using `LinearRouter`  
**Then** the response is `200` with body `42`

### AC4: SmartRouter matches the literal segment route (single route)

**Given** the default app uses `SmartRouter` and has only the route `/a/:id{[0-9]+}/[draft]` registered  
**When** `GET /a/42/[draft]` is requested  
**Then** `SmartRouter` selects `RegExpRouter` and the response is `200` with body `42`

### AC5: SmartRouter matches the literal segment route after fallback to TrieRouter

**Given** the default app uses `SmartRouter` and has `/a/:id{[0-9]+}/[draft]` plus unrelated routes registered  
**When** `GET /a/42/[draft]` is requested  
**Then** `SmartRouter` selects `TrieRouter` and the response is `200` with body `42`

### AC6: Non-matching paths remain unmatched

**Given** a route registered as `/a/:id{[0-9]+}/[draft]`  
**When** `GET /a/42/draft` (no brackets) is requested  
**Then** the route does not match  
**And when** `GET /a/42/[other]` is requested  
**Then** the route does not match

### AC7: Existing static segment routes still match

**Given** an existing route whose next segment after a regex parameter is `file.html` (or another literal without brackets)  
**When** a request matching that route is made  
**Then** the route matches as before the fix

### AC8: Regression tests cover TrieRouter and SmartRouter fallback

**Given** the fix is implemented  
**When** new tests are added to `src/router/trie-router/router.test.ts`  
**Then** the tests cover AC1, AC6, and AC7 for `TrieRouter`  
**And when** tests are added to `src/router/smart-router/router.test.ts`  
**Then** the tests cover AC4 and AC5 for `SmartRouter`

### AC9: Existing router test suites pass

**Given** the fix is implemented  
**When** the test suites for `RegExpRouter` and `LinearRouter` are run  
**Then** all tests pass without modification (verifying AC2, AC3, and no regressions)

### AC10: Test and lint pass

**Given** the fix and tests are implemented  
**When** `pnpm test` is run  
**Then** all tests pass  
**And when** `pnpm lint` is run  
**Then** no lint errors are reported

---

## Open Questions

None.

---

## References

- Ticket: PER-103
- Repo: `hono` (`pavithrachandran70/hono`)
- Base branch: `main`
