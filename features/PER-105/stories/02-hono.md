# lld-02-hono.md

**Story:** 02-hono  
**Ticket:** PER-107  
**Title:** Add ETag middleware regression tests for body cleanup  
**Type:** Story  
**Created:** 2025-01-22

---

## Overview

Add regression tests to verify that the ETag middleware correctly cancels discarded response bodies when returning `304 Not Modified` or computing an `ETag` from the body. No changes to middleware code are required — cleanup is automatic via the `Context.res` setter implemented in Story 1 (PER-106).

---

## Acceptance Criteria

### AC-1: ETag middleware cancels discarded bodies on 304 responses

**Given** a handler returns a response with a body  
**When** `etag()` middleware returns `304 Not Modified` and replaces `c.res`  
**Then** the original response body's stream is cancelled  
**And** the `cancel()` callback on the handler stream is invoked  
**And** the response has the same status (304), `ETag` header, and retained headers as current behavior

### AC-2: ETag middleware cancels clone branches when computing ETag from body

**Given** `etag()` middleware computes an `ETag` from `res.clone().body`  
**When** the middleware replaces `c.res` with a new response  
**Then** the original body branch is cancelled  
**And** readers on the original body report `done: true`  
**And** the response has the same `ETag` and headers as current behavior

---

## File Plan

### `src/middleware/etag/index.test.ts` — Regression tests for body cleanup

**Current state:** Does not exist in the repo tree. The repo-tree artifact cuts off at `src/helper/cookie/index.ts`, and the `src/middleware/` directory is not shown. The existence of `src/middleware/etag/index.ts` is assumed based on the ticket reference to the `etag()` middleware but cannot be confirmed from the provided repo-tree.

**Changes:** Create a new test file with 4 regression tests in a `describe('Response body cleanup')` block.

**File location:** `src/middleware/etag/index.test.ts` (path verified against full repository before implementation)

**Test cases:**

#### Test 1: `etag()` returns `304` when `If-None-Match` matches a preset `ETag`, and the original body is cancelled

**Setup:**
- Create a Hono app with the `etag()` middleware
- Create a handler that returns a response with a preset `ETag` header and a mock `ReadableStream` body
- The mock stream has a `cancel()` method that sets a flag when called
- Send a request with an `If-None-Match` header matching the preset `ETag`

**Assert:**
- The response status is `304 Not Modified`
- The `ETag` header matches the preset value
- The mock stream's `cancel()` method was called exactly once
- Retained headers (e.g. `Cache-Control`) are present in the 304 response

**Implementation sketch:**

```typescript
import { Hono } from '../../../hono'
import { etag } from './index'

describe('Response body cleanup', () => {
  it('returns 304 and cancels original body with preset ETag', async () => {
    const app = new Hono()
    let cancelCalled = false
    const mockStream = new ReadableStream({
      start(controller) {
        controller.enqueue(new TextEncoder().encode('test body'))
        controller.close()
      },
      cancel() {
        cancelCalled = true
      },
    })

    app.use('/test', etag())
    app.get('/test', (c) => {
      return new Response(mockStream, {
        headers: {
          'ETag': '"preset-etag-value"',
          'Cache-Control': 'max-age=3600',
        },
      })
    })

    const res = await app.request('/test', {
      headers: {
        'If-None-Match': '"preset-etag-value"',
      },
    })

    expect(res.status).toBe(304)
    expect(res.headers.get('ETag')).toBe('"preset-etag-value"')
    expect(res.headers.get('Cache-Control')).toBe('max-age=3600')
    expect(cancelCalled).toBe(true)
  })
})
```

**Notes:**
- Verifies `etag()` returns `304` when `If-None-Match` matches the preset `ETag`
- Mock stream's `cancel()` is called when `Context.res` setter replaces `c.res` with the `304` response
- Retained headers like `Cache-Control` are preserved in the `304` response

---

#### Test 2: `etag()` computes `ETag` from `res.clone().body`, returns the response with the `ETag` header, and the clone branch is cancelled

**Setup:**
- Create a Hono app with the `etag()` middleware
- Create a handler that returns a response with a mock `ReadableStream` body (no preset `ETag`)
- The mock stream has a `cancel()` method that sets a flag when called
- Send a request without an `If-None-Match` header

**Assert:**
- The response status is `200 OK` (not `304`)
- The `ETag` header is set (computed from the body)
- The mock stream's `cancel()` method was called exactly once (clone branch cancelled)
- The response body can still be read to completion

**Implementation sketch:**

```typescript
it('computes ETag from body and cancels clone branch', async () => {
  const app = new Hono()
  let cancelCalled = false
  const bodyContent = 'test body for etag computation'
  const mockStream = new ReadableStream({
    start(controller) {
      controller.enqueue(new TextEncoder().encode(bodyContent))
      controller.close()
    },
    cancel() {
      cancelCalled = true
    },
  })

  app.use('/test', etag())
  app.get('/test', (c) => {
    return new Response(mockStream)
  })

  const res = await app.request('/test')

  expect(res.status).toBe(200)
  expect(res.headers.get('ETag')).toBeTruthy()
  expect(cancelCalled).toBe(true) // Clone branch cancelled
  // Verify the body can still be read
  const text = await res.text()
  expect(text).toBe(bodyContent)
})
```

**Notes:**
- Verifies that when `etag()` computes an `ETag` from `res.clone().body`, the clone branch is cancelled when `c.res` is replaced
- The original body (passed to the final response) is not cancelled and can still be read to completion
- Confirms the `cancel()` callback fires on the clone branch (AC-2)

---

#### Test 3: `cancel()` callbacks fire on the handler stream when `304` is returned

**Setup:**
- Create a Hono app with the `etag()` middleware
- Create a handler that returns a response with a preset `ETag` and a mock `ReadableStream` body
- The mock stream has a `cancel()` method that increments a counter and sets a flag
- Send a request with an `If-None-Match` header matching the preset `ETag`

**Assert:**
- The response status is `304 Not Modified`
- The `cancel()` callback was invoked exactly once
- The flag set by the `cancel()` callback is `true`

**Implementation sketch:**

```typescript
it('cancel() callbacks fire on handler stream when 304 is returned', async () => {
  const app = new Hono()
  let cancelCallCount = 0
  let cancelCallbackFired = false
  const mockStream = new ReadableStream({
    start(controller) {
      controller.enqueue(new TextEncoder().encode('test body'))
      controller.close()
    },
    cancel() {
      cancelCallCount++
      cancelCallbackFired = true
    },
  })

  app.use('/test', etag())
  app.get('/test', (c) => {
    return new Response(mockStream, {
      headers: {
        'ETag': '"test-etag"',
      },
    })
  })

  const res = await app.request('/test', {
    headers: {
      'If-None-Match': '"test-etag"',
    },
  })

  expect(res.status).toBe(304)
  expect(cancelCallCount).toBe(1)
  expect(cancelCallbackFired).toBe(true)
})
```

**Notes:**
- Similar to Test 1 but explicitly focuses on verifying the `cancel()` callback fires
- Uses a counter to ensure the callback is invoked exactly once (not zero, not multiple times)
- Confirms the cleanup mechanism in `Context.res` setter correctly invokes `body.cancel()` on the discarded stream

---

#### Test 4: Status (`304`), `ETag` header, and retained headers match the current behavior exactly

**Setup:**
- Create a Hono app with the `etag()` middleware
- Create a handler that returns a response with a preset `ETag`, a body, and multiple headers (`Cache-Control`, `Content-Type`, `X-Custom-Header`)
- Send a request with an `If-None-Match` header matching the preset `ETag`

**Assert:**
- The response status is `304 Not Modified`
- The `ETag` header matches the preset value
- Retained headers (`Cache-Control`, `X-Custom-Header`) are present in the `304` response
- The `Content-Type` header is **not** present (per HTTP spec, `304` should not include entity headers unless required)
- The response body is empty (per HTTP spec, `304` has no body)

**Implementation sketch:**

```typescript
it('304 response has correct status, ETag, and retained headers', async () => {
  const app = new Hono()
  const mockStream = new ReadableStream({
    start(controller) {
      controller.enqueue(new TextEncoder().encode('test body'))
      controller.close()
    },
    cancel() {
      // No-op for this test
    },
  })

  app.use('/test', etag())
  app.get('/test', (c) => {
    return new Response(mockStream, {
      headers: {
        'ETag': '"test-etag"',
        'Cache-Control': 'max-age=3600',
        'Content-Type': 'text/plain',
        'X-Custom-Header': 'custom-value',
      },
    })
  })

  const res = await app.request('/test', {
    headers: {
      'If-None-Match': '"test-etag"',
    },
  })

  expect(res.status).toBe(304)
  expect(res.headers.get('ETag')).toBe('"test-etag"')
  expect(res.headers.get('Cache-Control')).toBe('max-age=3600')
  expect(res.headers.get('X-Custom-Header')).toBe('custom-value')
  // Content-Type should not be present in 304 response (per HTTP spec)
  expect(res.headers.get('Content-Type')).toBeNull()
  // Body should be empty
  const text = await res.text()
  expect(text).toBe('')
})
```

**Notes:**
- Ensures no regression in the `304` response construction by the `etag()` middleware
- Verifies status, `ETag`, and retained headers match expected behavior
- Checks that `Content-Type` is **not** included in the `304` response (per HTTP spec)
- Confirms the `304` response has an empty body (per HTTP spec)

---

### `src/middleware/etag/index.ts` — No changes

**Current state:** Assumed to contain the `etag()` middleware implementation. Path not confirmed in the provided repo-tree (cuts off at `src/helper/cookie/index.ts`). Verify path exists in full repository before implementation.

**Changes:** **None.** Cleanup is automatic via the `Context.res` setter implemented in Story 1. No middleware code changes are required.

---

## Interface Contracts

### `etag()` middleware behavior (unchanged)

**Before and After:**
- Returns `304 Not Modified` when `If-None-Match` matches the `ETag` (preset or computed)
- Computes `ETag` from `res.clone().body` when no preset `ETag` exists
- Retains headers (e.g. `Cache-Control`, `X-Custom-Header`) in the `304` response
- Does not include `Content-Type` or other entity headers in the `304` response (per HTTP spec)

**New guarantee (via Story 1):**
- The original response body is cancelled when `304` is returned
- The clone branch (from `res.clone().body`) is cancelled when the `ETag` is computed and `c.res` is replaced

---

## Algorithm Notes

### Test execution flow

1. **Test 1 (preset ETag, 304 response):**
   - Handler sets `c.res` to a response with a preset `ETag` and a mock stream
   - `etag()` middleware checks `If-None-Match` header
   - Middleware creates a new `304` response and assigns it to `c.res`
   - `Context.res` setter (Story 1) invokes `body.cancel()` on the original stream
   - Test asserts: `304` status, `ETag` header, retained headers, `cancel()` called

2. **Test 2 (computed ETag, clone branch cancelled):**
   - Handler sets `c.res` to a response with a mock stream (no preset `ETag`)
   - `etag()` middleware calls `res.clone().body` to compute the `ETag`
   - Middleware creates a new response with the computed `ETag` and assigns it to `c.res`
   - `Context.res` setter (Story 1) invokes `body.cancel()` on the clone branch
   - Test asserts: `200` status, `ETag` header set, `cancel()` called, original body still readable

3. **Test 3 (cancel callback fires):**
   - Same as Test 1, but explicitly tracks the `cancel()` callback invocation
   - Test asserts: `cancel()` callback fired exactly once

4. **Test 4 (no regression in 304 response):**
   - Handler sets `c.res` to a response with multiple headers
   - `etag()` middleware returns `304` and retains specific headers
   - Test asserts: `304` status, `ETag` header, retained headers present, `Content-Type` absent, body empty

### Edge cases

- **No preset `ETag`, body cannot be read:** If the body is already consumed or locked, `res.clone().body` may fail. This is not covered by this story (existing behavior)
- **`If-None-Match` header with multiple ETags:** The `etag()` middleware should handle this case correctly (existing behavior, not tested here unless regression is suspected)
- **`cancel()` throws an error:** The `Context.res` setter (Story 1) catches and suppresses the error. The `304` response is still returned. This is tested in Story 1, not here

---

## Tests

### Summary

Add **4 regression tests** in `src/middleware/etag/index.test.ts`:

1. `etag()` returns `304` when `If-None-Match` matches a preset `ETag`, and the original body is cancelled
2. `etag()` computes an `ETag` from `res.clone().body`, returns the response with the `ETag` header, and the clone branch is cancelled
3. `cancel()` callbacks fire on the handler stream when `304` is returned (verify by mocking the stream and asserting the callback was invoked)
4. Status (`304`), `ETag` header, and retained headers (e.g. `Cache-Control`) match the current behavior exactly (snapshot or explicit assertion)

**Exact test count:** 4 tests (matches breakdown.md Story 2 Verification Approach and requirements.md AC-1, AC-2)

---

## Verify

### Test coverage verification

- **AC-1 (ETag middleware cancels discarded bodies on 304 responses):** Covered by Test 1 (preset ETag, 304 returned, body cancelled) and Test 3 (cancel callback fires)
- **AC-2 (ETag middleware cancels clone branches when computing ETag from body):** Covered by Test 2 (computed ETag, clone branch cancelled, original body still readable)

### File path verification

- **`src/middleware/etag/index.test.ts`:** Path assumed based on ticket reference. **Not confirmed in repo-tree** (repo-tree cuts off at `src/helper/cookie/index.ts`). Verify path exists in full repository before implementation. ⚠️
- **`src/middleware/etag/index.ts`:** Path assumed based on middleware reference in ticket. **Not confirmed in repo-tree.** Verify path exists in full repository before implementation. ⚠️
- Both paths are concrete and follow expected Hono middleware structure

### Test count verification

- **breakdown.md:** Story 2 (PER-107) declares 4 tests
- **This LLD:** 4 tests (Test 1, Test 2, Test 3, Test 4)
- **requirements.md:** AC-1 and AC-2 both covered by the 4 tests
- Test count matches breakdown ✓

---

## Notes

- **No middleware code changes:** The cleanup is automatic via the `Context.res` setter implemented in Story 1 (PER-106). No changes to `src/middleware/etag/index.ts` are required
- **Test file creation:** `src/middleware/etag/index.test.ts` does not exist and will be created in the same directory as `src/middleware/etag/index.ts`. Verify the middleware directory path in the full repository before implementation
- **Repo-tree truncation:** The provided repo-tree cuts off at `src/helper/cookie/index.ts`, so the `src/middleware/` directory is not shown. The paths `src/middleware/etag/index.ts` and `src/middleware/etag/index.test.ts` are assumed based on the ticket reference to the `etag()` middleware but cannot be confirmed from the truncated repo-tree. **Verify these paths exist in the full repository before implementation.**
- **Test runner:** Use Vitest and `expect` (per repo-context/hono.md: `pnpm test`)
- **Mock stream implementation:** The mock `ReadableStream` with a `cancel()` callback may need to be constructed carefully to work across all Hono runtimes (Bun, Deno, Node.js, Cloudflare Workers). If runtime-specific issues arise, adjust the mock or use a test helper
- **Retained headers in 304 response:** Per HTTP spec, a `304` response should include certain headers (e.g. `Cache-Control`, `ETag`) but not entity headers (e.g. `Content-Type`, `Content-Length`). Test 4 verifies this behavior
- **Clone branch cancellation:** Test 2 verifies that the clone branch (from `res.clone().body`) is cancelled when the `ETag` is computed and `c.res` is replaced. The original body (passed to the final response) is not cancelled and can still be read

---

## Dependencies

- **Story 1 (PER-106 — Implement core response body cleanup mechanism in Context):** This story depends on Story 1 being complete and all tests passing. The cleanup mechanism in `Context.res` setter must exist before these regression tests can verify it
