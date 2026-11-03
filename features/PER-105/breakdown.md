# breakdown.md

**Ticket:** PER-105  
**Title:** Release the discarded response body when middleware replaces `c.res`  
**Type:** Epic  
**Created:** 2025-01-22

---

## Overview

This breakdown covers the implementation of automatic response body cleanup when middleware replaces `c.res`, preventing stream leaks and unbounded buffering across the Hono framework. The work is split into five implementation stories executed in sequence: core cleanup mechanism, ETag middleware fixes, PrettyJSON middleware fixes, verification of other middleware, and comprehensive regression tests.

---

## Execution Order

Stories must be implemented in the order listed below. Each story builds on the previous one and must pass all tests before the next story begins.

---

### Story 1: Implement core response body cleanup mechanism in Context

**Repo:** `hono`

**Description:**

Extend the `Context.res` setter in `src/context.ts` to automatically cancel the previous response body when `c.res` is replaced with a new `Response`. The cleanup logic must:
- Check if a previous response exists (`this._res`)
- Check if the previous body is different from the new body (reference equality: `this._res.body !== res.body`)
- Check if the previous body exists (`this._res.body`) before invoking `cancel()`
- Invoke `this._res.body.cancel()` when all conditions are true
- Wrap the `cancel()` call in a try/catch block, logging errors in development and silently ignoring them in production
- Assign the new response to `this._res` regardless of cleanup success

This story establishes the core contract for all middleware that replace `c.res` and implements AC-7 (header merging preserved) by keeping the existing assignment logic unchanged.

**Covers:**
- AC-5 (Reused bodies continue to stream — reference equality check skips cleanup)
- AC-6 (Failed body release does not break request processing — try/catch error handling)
- AC-7 (Header merging preserved — assignment logic unchanged, implementation)

**Dependencies:** None (this is the foundation story)

**Allowed paths:**
- `src/context.ts` — Modify the `res` setter to add cleanup logic
- `src/context.test.ts` — Add unit tests for cleanup behavior

**Verification Approach:**

Add **4 unit tests** in `src/context.test.ts`:
1. `c.res` replacement cancels the previous body when body references differ
2. `c.res` replacement skips cleanup when the same body is reused (reference equality)
3. Cleanup error (thrown by `body.cancel()`) does not prevent the new response from being assigned
4. Cleanup error does not throw or reject at the `Context` level (error is caught and suppressed)

Run `pnpm test` to verify the cleanup mechanism works correctly in isolation before proceeding to middleware tests.

**Exact test count:** 4 tests

**Notes:**
- The cleanup is best-effort: if `body.cancel()` is unavailable or throws, the error is trapped and the new response is still assigned
- The reference equality check (`previousBody === newBody`) assumes middleware that reuse a body pass the same object reference
- Error logging in development uses `process.env.NODE_ENV === 'development'` (or equivalent runtime check) — silent ignore in production
- The explicit check `&& this._res.body` ensures `cancel()` is only called when the body exists, avoiding optional chaining complexity

---

### Story 2: Add ETag middleware regression tests for body cleanup

**Repo:** `hono`

**Description:**

Add regression tests to verify that the ETag middleware correctly cancels discarded response bodies when returning `304 Not Modified` or computing an `ETag` from the body. The tests must confirm:
- The original body is cancelled when a preset `ETag` matches and `304` is returned
- The clone branch is cancelled when an `ETag` is computed from `res.clone().body`
- `cancel()` callbacks fire on the handler stream
- Status, `ETag`, and headers match current behavior (no regression)

No changes to ETag middleware code are required — the cleanup is automatic via the `Context.res` setter implemented in Story 1.

**Covers:**
- AC-1 (ETag middleware cancels discarded bodies on 304 responses)
- AC-2 (ETag middleware cancels clone branches when computing ETag from body)

**Dependencies:**
- `after = [1]` (Story 1 must complete first — the cleanup mechanism must exist)

**Allowed paths:**
- `src/middleware/etag/index.test.ts` — Add or extend test cases for body cleanup (create if missing)
- No changes to `src/middleware/etag/index.ts` (cleanup is automatic)

**Verification Approach:**

Add **4 regression tests**:
1. `etag()` returns `304` when `If-None-Match` matches a preset `ETag`, and the original body is cancelled
2. `etag()` computes an `ETag` from `res.clone().body`, returns the response with the `ETag` header, and the clone branch is cancelled
3. `cancel()` callbacks fire on the handler stream when `304` is returned (verify by mocking the stream and asserting the callback was invoked)
4. Status (`304`), `ETag` header, and retained headers (e.g. `Cache-Control`) match the current behavior exactly (snapshot or explicit assertion)

Run `pnpm test` to verify ETag cleanup behavior.

**Exact test count:** 4 tests

**Notes:**
- The test for `cancel()` callbacks (test 3) may require mocking `ReadableStream` or using a test helper to observe the callback
- The test for retained headers (test 4) ensures no regression in the `304` response construction
- Test file location: confirm `src/middleware/etag/` exists in the full repository before implementation (not visible in truncated repo-tree)

---

### Story 3: Add PrettyJSON middleware regression tests for body cleanup

**Repo:** `hono`

**Description:**

Add regression tests to verify that the PrettyJSON middleware correctly cancels the original response body after formatting JSON, and passes through `204` and non-JSON responses unchanged. The tests must confirm:
- The original body is cancelled after `prettyJSON()` formats JSON
- `204` responses pass through unchanged (no body, no cancel)
- Non-JSON responses pass through unchanged (no clone, no cancel)
- Formatted JSON output, status, and headers match current behavior (no regression)

No changes to PrettyJSON middleware code are required — the cleanup is automatic via the `Context.res` setter implemented in Story 1.

**Covers:**
- AC-3 (PrettyJSON middleware cancels original body after formatting)
- AC-4 (PrettyJSON middleware preserves non-JSON and empty responses)

**Dependencies:**
- `after = [1]` (Story 1 must complete first — the cleanup mechanism must exist)

**Allowed paths:**
- `src/middleware/pretty-json/index.test.ts` — Add or extend test cases for body cleanup (create if missing)
- No changes to `src/middleware/pretty-json/index.ts` (cleanup is automatic)

**Verification Approach:**

Add **4 regression tests**:
1. `prettyJSON()` formats a JSON response, and the original body is cancelled (verify by mocking the body or observing the `cancel()` call)
2. `prettyJSON()` passes through a `204 No Content` response unchanged (no body cloned, no cancel invoked)
3. `prettyJSON()` passes through a non-JSON response (e.g. `text/html`) unchanged (no body cloned, no cancel invoked)
4. The formatted JSON output, status, and headers match the current behavior exactly (snapshot or explicit assertion)

Run `pnpm test` to verify PrettyJSON cleanup behavior.

**Exact test count:** 4 tests

**Notes:**
- The test for cancelled body (test 1) may require mocking `ReadableStream` or using a test helper to observe the `cancel()` call
- The passthrough tests (tests 2 and 3) ensure no regression when `prettyJSON()` does not transform the response
- Test file location: confirm `src/middleware/pretty-json/` exists in the full repository before implementation (not visible in truncated repo-tree)

---

### Story 4: Verify cleanup behavior in other middleware (cache, compress, timeout, jsx-renderer, secure-headers)

**Repo:** `hono`

**Description:**

Manually verify that the automatic body cleanup does not break existing behavior in other middleware that replace `c.res`: `cache`, `compress`, `timeout`, `jsx-renderer`, and `secure-headers`. For each middleware, run its existing test suite (if any) and confirm that status, headers, and body transformations are unchanged. If a middleware has no tests, perform a manual smoke test by running the middleware in a representative scenario and inspecting the response.

No changes to middleware code are required — the cleanup is automatic via the `Context.res` setter implemented in Story 1. This story verifies AC-7 (header merging preserved) across all affected middleware.

**Covers:**
- AC-7 (Header merging preserved — manual verification that `set-cookie` and other headers are merged correctly across all middleware)

**Dependencies:**
- `after = [1]` (Story 1 must complete first — the cleanup mechanism must exist)

**Allowed paths:**
- No file changes (verification only)
- Observed paths (read-only): `src/middleware/cache/`, `src/middleware/compress/`, `src/middleware/timeout/`, `src/middleware/jsx-renderer/`, `src/middleware/secure-headers/`

**Verification Approach:**

For each middleware, perform the following:
1. Run the existing test suite for the middleware (if it exists) using `pnpm test`
2. If no test suite exists, manually test the middleware in a representative scenario:
   - `cache`: Verify cache hits return the cached response with correct headers
   - `compress`: Verify compressed responses have correct `Content-Encoding` and body
   - `timeout`: Verify timeout responses return `504` with correct headers
   - `jsx-renderer`: Verify JSX renders to HTML with correct `Content-Type`
   - `secure-headers`: Verify security headers are added to the response
3. Inspect the response to confirm status, headers, and body match expected behavior
4. Document the verification results in the story's Notes (or in a PR comment)

No new tests are added unless current behavior is incorrect. If a regression is found, investigate whether it is caused by the cleanup mechanism or by a pre-existing bug.

**Exact test count:** 0 tests (manual verification only)

**Notes:**
- This story gates the merge: if any middleware shows a regression, investigate and fix before proceeding to Story 5
- If a middleware has a comprehensive test suite, running the tests is sufficient verification
- If a middleware has no tests, the manual smoke test should cover the most common use case (e.g. cache hit, compression enabled, timeout triggered, JSX render, headers added)
- Middleware paths: confirm these directories exist in the full repository before implementation (not visible in truncated repo-tree)

---

### Story 5: Add comprehensive regression test suite for all cleanup scenarios

**Repo:** `hono`

**Description:**

Add a comprehensive regression test suite to `src/context.test.ts` covering all cleanup scenarios across middleware and edge cases. This suite acts as the final gate before merge, verifying that the cleanup mechanism works correctly in all documented scenarios and does not introduce regressions. The tests must cover:
- ETag middleware: both preset `ETag` (304) and computed `ETag` (from body) cases
- PrettyJSON middleware: JSON formatting, `204` passthrough, non-JSON passthrough
- Reused body: middleware that reuse the same body reference (no cancel)
- Failed cleanup: `body.cancel()` throws or rejects, and the new response is still returned
- Header merging: `set-cookie` headers are appended (not overwritten) when `c.res` is replaced
- Multi-runtime compatibility: tests run on Bun, Deno, Node.js, and Cloudflare Workers (via the existing test infrastructure)

This suite consolidates the test coverage from Stories 1–4 and adds any missing edge cases.

**Covers:**
- AC-1 (ETag 304)
- AC-2 (ETag computed from body)
- AC-3 (PrettyJSON formatting)
- AC-4 (PrettyJSON passthrough)
- AC-5 (Reused body)
- AC-6 (Failed cleanup)
- AC-7 (Header merging)

**Dependencies:**
- `after = [1, 2, 3, 4]` (all prior stories must complete first — this is the final verification)

**Allowed paths:**
- `src/context.test.ts` — Add comprehensive test suite (or create `src/context.cleanup.test.ts` if the existing file is large)
- No changes to middleware files (tests only)

**Verification Approach:**

Add **12 comprehensive tests** covering all scenarios:

1. **ETag preset 304:** `etag()` returns `304` with a preset `ETag`, original body is cancelled, status and headers match current behavior
2. **ETag computed from body:** `etag()` computes `ETag` from `res.clone().body`, clone branch is cancelled, `ETag` header is set, status and headers match current behavior
3. **PrettyJSON formats JSON:** `prettyJSON()` formats JSON, original body is cancelled, formatted output matches current behavior
4. **PrettyJSON passes through 204:** `prettyJSON()` does not transform `204`, no body is cloned or cancelled
5. **PrettyJSON passes through non-JSON:** `prettyJSON()` does not transform `text/html`, no body is cloned or cancelled
6. **Reused body streams to end:** Middleware replaces `c.res` with a response that reuses the same body reference, body streams to completion without cancellation
7. **Failed cleanup does not break request:** `body.cancel()` throws an error, the new response is still assigned to `c.res` and returned
8. **Cleanup error logged in development:** `body.cancel()` throws an error, the error is logged to the console in development mode
9. **Cleanup error silent in production:** `body.cancel()` throws an error, no error is logged in production mode
10. **Header merging appends set-cookie:** Middleware replaces `c.res` with a new response, `set-cookie` headers from both responses are appended (not overwritten)
11. **Multi-runtime Bun:** The cleanup mechanism works correctly on Bun (run existing test suite on Bun runtime)
12. **Multi-runtime Deno/Node.js/Cloudflare Workers:** The cleanup mechanism works correctly on other runtimes (run existing test suite on each runtime)

Run `pnpm test` on all supported runtimes to verify the cleanup mechanism works correctly across the board.

**Exact test count:** 12 tests

**Notes:**
- Tests 11 and 12 may be covered by the existing CI infrastructure (e.g. GitHub Actions matrix testing on multiple runtimes) — if so, document that the cleanup tests pass on all runtimes rather than adding new runtime-specific tests
- Test 10 (header merging) ensures no regression in the existing `c.res` assignment logic
- Tests 7–9 (error handling) confirm that cleanup errors do not break request processing, meeting AC-6
- This suite is the final gate before merge: all 12 tests must pass, and `pnpm lint` must pass with no errors

---

## Summary

| Story | Title | Repo | Depends On | Test Count |
|---|---|---|---|---|
| 1 | Implement core response body cleanup mechanism in Context | `hono` | — | 4 |
| 2 | Add ETag middleware regression tests for body cleanup | `hono` | 1 | 4 |
| 3 | Add PrettyJSON middleware regression tests for body cleanup | `hono` | 1 | 4 |
| 4 | Verify cleanup behavior in other middleware | `hono` | 1 | 0 (manual) |
| 5 | Add comprehensive regression test suite for all cleanup scenarios | `hono` | 1, 2, 3, 4 | 12 |

**Total test count:** 24 tests (4 + 4 + 4 + 0 + 12)

---

## Dependencies

- **Story 1** is the foundation: it implements the cleanup mechanism in `Context.res` setter and AC-7 (header merging preserved) via unchanged assignment logic
- **Stories 2, 3, 4** depend on Story 1 (the mechanism must exist before testing middleware)
- **Story 5** depends on all prior stories (it consolidates and verifies the entire implementation)

No story may begin until its dependencies are complete and all tests pass.

---

## Verification Summary

All acceptance criteria are covered by the test suites:

- **AC-1 (ETag 304):** Story 2 test 1, Story 5 test 1
- **AC-2 (ETag computed from body):** Story 2 test 2, Story 5 test 2
- **AC-3 (PrettyJSON formatting):** Story 3 test 1, Story 5 test 3
- **AC-4 (PrettyJSON passthrough):** Story 3 tests 2 and 3, Story 5 tests 4 and 5
- **AC-5 (Reused body):** Story 1 test 2, Story 5 test 6
- **AC-6 (Failed cleanup):** Story 1 tests 3 and 4, Story 5 tests 7, 8, and 9
- **AC-7 (Header merging):** Story 1 (implementation preserves assignment logic), Story 4 (manual verification across all middleware), Story 5 test 10

The total test count (24 tests) matches the verification approach across all stories. Each story's test count is listed in the Verification Approach section and summarized in the table above.

---

## Notes

- No changes to middleware code are required — the cleanup is automatic via the `Context.res` setter
- All middleware that replace `c.res` benefit from the cleanup without code changes
- The cleanup mechanism is best-effort: errors are trapped and do not break request processing
- The reference equality check (`previousBody === newBody`) assumes middleware that reuse a body pass the same object reference — this is verified in Story 1 and Story 5
- Story 4 is a manual verification gate: if any middleware shows a regression, investigate and fix before proceeding to Story 5
- The pull request must be opened on `pavithrachandran70/hono`, not `honojs/hono`, as stated in the ticket
- Middleware paths (`src/middleware/etag/`, `src/middleware/pretty-json/`, `src/middleware/cache/`, etc.) are not listed in the truncated repo-tree but are assumed to exist in the full repository — verify before implementation
- AC-7 (header merging preserved) is covered by Story 1's implementation (unchanged assignment logic) and Story 4's manual verification (all middleware tested for header behavior)
