# requirements.md

**Ticket:** PER-105  
**Title:** Release the discarded response body when middleware replaces `c.res`  
**Type:** Epic

---

## Overview

When middleware replaces `c.res` with a new `Response`, the previous response body must be properly released. Currently, stream `cancel()` callbacks do not run, and unread cloned bodies remain buffered indefinitely. This affects multiple middleware components (`etag`, `prettyJSON`, `cache`, `compress`, `timeout`, `jsx-renderer`, `secure-headers`) that replace `c.res`.

---

## Acceptance Criteria

### AC-1: ETag middleware cancels discarded bodies on 304 responses

**Given** a handler returns a response with a body  
**When** `etag()` middleware returns `304 Not Modified` and replaces `c.res`  
**Then** the original response body's stream is cancelled  
**And** the `cancel()` callback on the handler stream is invoked  
**And** the response has the same status (304), `ETag` header, and retained headers as current behavior

**Verification:** 
- Story 1 (PER-106) implements the core cleanup mechanism in `src/context.ts` (1 test verifying cleanup logic for reused vs. replaced bodies)
- Story 2 (PER-107) adds regression tests in `src/middleware/etag/index.test.ts` (2 tests: preset ETag 304 case, and cancel callback fires)
- Story 5 (PER-110) adds comprehensive test in `src/context.test.ts` (1 test: ETag preset 304 consolidation)

**Total tests covering AC-1:** 4 tests

---

### AC-2: ETag middleware cancels clone branches when computing ETag from body

**Given** `etag()` middleware computes an `ETag` from `res.clone().body`  
**When** the middleware replaces `c.res` with a new response  
**Then** the original body branch is cancelled  
**And** readers on the original body report `done: true`  
**And** the response has the same `ETag` and headers as current behavior

**Verification:**
- Story 1 (PER-106) implements the core cleanup mechanism in `src/context.ts` (same test as AC-1)
- Story 2 (PER-107) adds regression tests in `src/middleware/etag/index.test.ts` (2 tests: computed ETag from body, status/headers match)
- Story 5 (PER-110) adds comprehensive test in `src/context.test.ts` (1 test: ETag computed from body consolidation)

**Total tests covering AC-2:** 4 tests

---

### AC-3: PrettyJSON middleware cancels original body after formatting

**Given** a handler returns a JSON response  
**When** `prettyJSON()` reads `c.res.clone().json()` and replaces `c.res` with formatted JSON  
**Then** the original response body is cancelled  
**And** the formatted response has the same JSON content, status, and headers as current behavior

**Verification:**
- Story 1 (PER-106) implements the core cleanup mechanism in `src/context.ts` (same test as AC-1, AC-2)
- Story 3 (PER-108) adds regression tests in `src/middleware/pretty-json/index.test.ts` (2 tests: JSON formatted and body cancelled, output/headers match)
- Story 5 (PER-110) adds comprehensive test in `src/context.test.ts` (1 test: PrettyJSON formatting consolidation)

**Total tests covering AC-3:** 4 tests

---

### AC-4: PrettyJSON middleware preserves non-JSON and empty responses

**Given** a handler returns a `204` or a non-JSON response  
**When** `prettyJSON()` processes the response  
**Then** the response passes through unchanged  
**And** no body is cancelled (none exists or none was cloned)

**Verification:**
- Story 3 (PER-108) adds regression tests in `src/middleware/pretty-json/index.test.ts` (2 tests: 204 passthrough, non-JSON passthrough)
- Story 5 (PER-110) adds comprehensive tests in `src/context.test.ts` (2 tests: 204 passthrough consolidation, non-JSON passthrough consolidation)

**Total tests covering AC-4:** 4 tests

---

### AC-5: Reused bodies continue to stream

**Given** middleware replaces `c.res` with a response that reuses the previous body  
**When** the response is streamed to completion  
**Then** the body streams to the end without cancellation

**Verification:**
- Story 1 (PER-106) implements the reference equality check in `src/context.ts` and adds unit test in `src/context.test.ts` (1 test: reused body skips cleanup)
- Story 5 (PER-110) adds comprehensive test in `src/context.test.ts` (1 test: reused body streams to end consolidation)

**Total tests covering AC-5:** 2 tests

---

### AC-6: Failed body release does not break request processing

**Given** an error occurs while releasing the old response body  
**When** middleware replaces `c.res`  
**Then** the new response is still returned  
**And** the request does not return a `500` error

**Verification:**
- Story 1 (PER-106) implements try/catch error handling in `src/context.ts` and adds unit tests in `src/context.test.ts` (2 tests: cleanup error does not prevent assignment, error does not propagate)
- Story 5 (PER-110) adds comprehensive tests in `src/context.test.ts` (3 tests: failed cleanup does not break request, error logged in dev, error silent in prod)

**Total tests covering AC-6:** 5 tests

---

### AC-7: Header merging behavior is preserved

**Given** middleware replaces `c.res`  
**When** headers from the old and new responses are merged  
**Then** `set-cookie` and other headers are merged as they are today  
**And** header merging behavior matches current implementation

**Verification:**
- Story 1 (PER-106) preserves existing assignment logic in `src/context.ts` (implementation review confirms no change to header merging)
- Story 4 (PER-109) manually verifies header merging in all affected middleware (`cache`, `compress`, `timeout`, `jsx-renderer`, `secure-headers`) (0 tests, manual verification documented)
- Story 5 (PER-110) adds comprehensive test in `src/context.test.ts` (1 test: header merging appends set-cookie)

**Total tests covering AC-7:** 1 test (plus manual verification in Story 4)

---

## Verification Summary

**Total regression tests across all stories:**
- Story 1 (PER-106): 4 tests (`src/context.test.ts` — core cleanup mechanism unit tests)
- Story 2 (PER-107): 4 tests (`src/middleware/etag/index.test.ts` — ETag middleware regression tests)
- Story 3 (PER-108): 4 tests (`src/middleware/pretty-json/index.test.ts` — PrettyJSON middleware regression tests)
- Story 4 (PER-109): 0 tests (manual verification only — documented in PR comment)
- Story 5 (PER-110): 12 tests (`src/context.test.ts` — comprehensive test suite consolidating all scenarios)

**Total test count:** 24 tests (4 + 4 + 4 + 0 + 12)

**File paths verified against repo-tree:**
- `src/context.ts` — exists ✓
- `src/context.test.ts` — exists ✓
- `src/middleware/etag/index.test.ts` — will be created in Story 2 (does not exist in truncated repo-tree)
- `src/middleware/pretty-json/index.test.ts` — will be created in Story 3 (does not exist in truncated repo-tree)

**Note:** Middleware paths (`src/middleware/etag/`, `src/middleware/pretty-json/`, `src/middleware/cache/`, etc.) are not listed in the provided repo-tree, which cuts off at `src/helper/cookie/index.ts`. These paths are assumed to exist in the full repository and must be verified before implementation.

---

## Additional Verification

- All tests in `pnpm test` pass (verified by CI on Bun, Deno, Node.js, Cloudflare Workers)
- All linting checks in `pnpm lint` pass
- No changes to public middleware options or `Context` method signatures
- Pull request opened on `pavithrachandran70/hono`, not `honojs/hono`

---

## Out of Scope

- Cleanup behavior for middleware not listed (`cache`, `compress`, `timeout`, `jsx-renderer`, `secure-headers`) is verified for consistency in Story 4 but not regression-tested unless current behavior is incorrect
- Framework, language, ORM, or cloud stack selection (Hono's existing TypeScript + multi-runtime stack is retained)
- Changes to middleware public APIs or `Context` method signatures (cleanup is internal to `Context.res` setter)
- Request body cleanup, WebSocket cleanup, or other resource release scenarios beyond response body cleanup

---

## Notes

- This fix addresses resource cleanup across multiple middleware components that replace `c.res`
- The cleanup contract ensures stream cancellation and clone cleanup work consistently without breaking existing middleware behavior
- Error handling during cleanup is best-effort: errors are trapped and do not prevent the new response from being returned
- The reference equality check (`previousBody === newBody`) assumes middleware that reuse a body pass the same object reference — middleware that wrap or reconstruct the body may trigger false positives (deferred to future work if needed)
- All concrete file paths (`src/context.ts`, `src/context.test.ts`) are verified against the repo-tree; middleware paths are assumed to exist in the full repository (repo-tree appears truncated)
