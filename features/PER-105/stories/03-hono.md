# lld-03-hono.md

**Story:** 03-hono  
**Ticket:** PER-108  
**Title:** Add PrettyJSON middleware regression tests for body cleanup  
**Type:** Story  
**Created:** 2025-01-22

---

## Overview

Add regression tests to verify that the PrettyJSON middleware correctly cancels the original response body after formatting JSON, and passes through `204` and non-JSON responses unchanged. No changes to middleware code are required — cleanup is automatic via the `Context.res` setter implemented in Story 1 (PER-106).

---

## Acceptance Criteria

### AC-3: PrettyJSON middleware cancels original body after formatting

**Given** a handler returns a JSON response  
**When** `prettyJSON()` reads `c.res.clone().json()` and replaces `c.res` with formatted JSON  
**Then** the original response body is cancelled  
**And** the formatted response has the same JSON content, status, and headers as current behavior

### AC-4: PrettyJSON middleware preserves non-JSON and empty responses

**Given** a handler returns a `204` or a non-JSON response  
**When** `prettyJSON()` processes the response  
**Then** the response passes through unchanged  
**And** no body is cancelled (none exists or none was cloned)

---

## File Plan

### `src/middleware/pretty-json/index.test.ts` — Regression tests for body cleanup

**Current state:** Does not exist in the provided repo-tree. The repo-tree artifact cuts off at `src/helper/cookie/index.ts`, and the `src/middleware/` directory is not shown. The existence of `src/middleware/pretty-json/index.ts` is assumed based on the ticket reference to the `prettyJSON()` middleware but cannot be confirmed from the provided repo-tree.

**Changes:** Create a new test file with 4 regression tests in a `describe('Response body cleanup')` block.

**File location:** `src/middleware/pretty-json/index.test.ts` (path verified against full repository before implementation)

**Test cases:**

#### Test 1: `prettyJSON()` formats a JSON response, and the original body is cancelled

**Setup:**
- Create a Hono app with the `prettyJSON()` middleware.
- Create a handler that returns a JSON response with a mock `ReadableStream` body.
- The mock stream has a `cancel()` method that sets a flag when called.
- Send a request to the handler.

**Assert:**
- The response status matches the original status (e.g. `200 OK`).
- The response body contains formatted JSON (with indentation).
- The mock stream's `cancel()` method was called exactly once.
- Response headers match current behavior (e.g. `Content-Type: application/json`).

**Implementation sketch:**

```typescript
import { Hono } from '../../hono'
import { prettyJSON } from './index'

describe('Response body cleanup', () => {
  it('formats JSON and cancels original body', async () => {
    const app = new Hono()
    let cancelCalled = false
    const jsonData = { message: 'test', nested: { value: 42 } }
    
    app.use('/test', prettyJSON())
    app.get('/test', (c) => {
      const mockStream = new ReadableStream({
        start(controller) {
          controller.enqueue(new TextEncoder().encode(JSON.stringify(jsonData)))
          controller.close()
        },
        cancel() {
          cancelCalled = true
        },
      })
      return new Response(mockStream, {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
      })
    })

    const res = await app.request('/test')

    expect(res.status).toBe(200)
    expect(res.headers.get('Content-Type')).toBe('application/json')
    expect(cancelCalled).toBe(true)
    
    const text = await res.text()
    // Verify formatted JSON (indented)
    expect(text).toContain('  "message": "test"')
    expect(text).toContain('  "nested": {')
    expect(text).toContain('    "value": 42')
  })
})
```

**Notes:**
- The test verifies that `prettyJSON()` formats the JSON response and cancels the original body.
- The mock stream's `cancel()` method is called when `Context.res` setter replaces `c.res` with the formatted response.
- The formatted JSON output is verified by checking for indentation (default is 2 spaces).

---

#### Test 2: `prettyJSON()` passes through a `204 No Content` response unchanged (no body cloned, no cancel invoked)

**Setup:**
- Create a Hono app with the `prettyJSON()` middleware.
- Create a handler that returns a `204 No Content` response (no body).
- Send a request to the handler.

**Assert:**
- The response status is `204 No Content`.
- No body is present (or body is empty).
- No `cancel()` method is called (since no body exists).
- Response headers match current behavior.

**Implementation sketch:**

```typescript
it('passes through 204 No Content unchanged', async () => {
  const app = new Hono()
  let cancelCalled = false

  app.use('/test', prettyJSON())
  app.get('/test', (c) => {
    // 204 has no body
    return new Response(null, {
      status: 204,
      headers: { 'X-Custom-Header': 'test-value' },
    })
  })

  const res = await app.request('/test')

  expect(res.status).toBe(204)
  expect(res.headers.get('X-Custom-Header')).toBe('test-value')
  expect(cancelCalled).toBe(false) // No cancel expected
  
  const text = await res.text()
  expect(text).toBe('') // No body
})
```

**Notes:**
- The test verifies that `prettyJSON()` does not transform a `204` response.
- Since `204` has no body, no clone is made and no `cancel()` is called.
- Custom headers are preserved.

---

#### Test 3: `prettyJSON()` passes through a non-JSON response (e.g. `text/html`) unchanged (no body cloned, no cancel invoked)

**Setup:**
- Create a Hono app with the `prettyJSON()` middleware.
- Create a handler that returns an HTML response (`Content-Type: text/html`).
- Send a request to the handler.

**Assert:**
- The response status matches the original status (e.g. `200 OK`).
- The response body is unchanged (HTML, not JSON).
- No `cancel()` method is called (since the body is not cloned for non-JSON responses).
- Response headers match current behavior.

**Implementation sketch:**

```typescript
it('passes through non-JSON response unchanged', async () => {
  const app = new Hono()
  let cancelCalled = false
  const htmlContent = '<html><body>Hello World</body></html>'

  app.use('/test', prettyJSON())
  app.get('/test', (c) => {
    const mockStream = new ReadableStream({
      start(controller) {
        controller.enqueue(new TextEncoder().encode(htmlContent))
        controller.close()
      },
      cancel() {
        cancelCalled = true
      },
    })
    return new Response(mockStream, {
      status: 200,
      headers: { 'Content-Type': 'text/html' },
    })
  })

  const res = await app.request('/test')

  expect(res.status).toBe(200)
  expect(res.headers.get('Content-Type')).toBe('text/html')
  expect(cancelCalled).toBe(false) // No cancel expected for non-JSON
  
  const text = await res.text()
  expect(text).toBe(htmlContent)
})
```

**Notes:**
- The test verifies that `prettyJSON()` does not transform non-JSON responses.
- Since the `Content-Type` is not `application/json`, the middleware should pass the response through unchanged.
- No clone is made, so no `cancel()` is called.

---

#### Test 4: The formatted JSON output, status, and headers match the current behavior exactly

**Setup:**
- Create a Hono app with the `prettyJSON()` middleware.
- Create a handler that returns a JSON response with multiple headers.
- Send a request to the handler.

**Assert:**
- The response status matches the original status.
- The response body contains formatted JSON (with indentation).
- Response headers match current behavior (including custom headers).
- The JSON structure is preserved (no data loss).

**Implementation sketch:**

```typescript
it('formatted JSON output, status, and headers match current behavior', async () => {
  const app = new Hono()
  const jsonData = { 
    users: [
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ],
    count: 2 
  }

  app.use('/test', prettyJSON())
  app.get('/test', (c) => {
    return c.json(jsonData, 200, {
      'X-Custom-Header': 'custom-value',
      'Cache-Control': 'max-age=3600',
    })
  })

  const res = await app.request('/test')

  expect(res.status).toBe(200)
  expect(res.headers.get('Content-Type')).toBe('application/json; charset=UTF-8')
  expect(res.headers.get('X-Custom-Header')).toBe('custom-value')
  expect(res.headers.get('Cache-Control')).toBe('max-age=3600')
  
  const text = await res.text()
  // Verify formatted JSON (indented)
  expect(text).toContain('  "users": [')
  expect(text).toContain('    {')
  expect(text).toContain('      "id": 1,')
  expect(text).toContain('      "name": "Alice"')
  expect(text).toContain('  "count": 2')
  
  // Verify JSON structure is preserved
  const parsed = JSON.parse(text)
  expect(parsed).toEqual(jsonData)
})
```

**Notes:**
- The test verifies that `prettyJSON()` formats the JSON response with indentation (default 2 spaces).
- Custom headers are preserved in the formatted response.
- The JSON structure is preserved (no data loss during formatting).
- This test ensures no regression in the formatting logic.

---

### `src/middleware/pretty-json/index.ts` — No changes

**Current state:** Assumed to contain the `prettyJSON()` middleware implementation. Path not confirmed in the provided repo-tree (cuts off at `src/helper/cookie/index.ts`). Verify path exists in full repository before implementation.

**Changes:** **None.** Cleanup is automatic via the `Context.res` setter implemented in Story 1. No middleware code changes are required.

---

## Interface Contracts

### `prettyJSON()` middleware behavior (unchanged)

**Before and After:**
- Formats JSON responses with indentation (default 2 spaces).
- Passes through `204 No Content` responses unchanged (no body).
- Passes through non-JSON responses unchanged (e.g. `text/html`).
- Preserves status, headers, and JSON structure.

**New guarantee (via Story 1):**
- The original response body is cancelled when `prettyJSON()` replaces `c.res` with the formatted response.
- No body is cancelled for `204` or non-JSON responses (no clone is made).

---

## Algorithm Notes

### Test execution flow

1. **Test 1 (JSON formatting, body cancelled):**
   - Handler sets `c.res` to a JSON response with a mock stream.
   - `prettyJSON()` middleware calls `c.res.clone().json()` to read the JSON data.
   - Middleware creates a new response with formatted JSON and assigns it to `c.res`.
   - `Context.res` setter (Story 1) invokes `body.cancel()` on the original stream.
   - Test asserts: status, `Content-Type`, `cancel()` called, formatted JSON output.

2. **Test 2 (204 passthrough, no cancel):**
   - Handler sets `c.res` to a `204 No Content` response (no body).
   - `prettyJSON()` middleware detects no body and passes the response through unchanged.
   - No clone is made, so no `cancel()` is called.
   - Test asserts: status `204`, no body, no `cancel()` called.

3. **Test 3 (non-JSON passthrough, no cancel):**
   - Handler sets `c.res` to an HTML response (`Content-Type: text/html`).
   - `prettyJSON()` middleware detects non-JSON `Content-Type` and passes the response through unchanged.
   - No clone is made, so no `cancel()` is called.
   - Test asserts: status, `Content-Type: text/html`, no `cancel()` called, HTML body unchanged.

4. **Test 4 (no regression in formatting):**
   - Handler sets `c.res` to a JSON response with custom headers.
   - `prettyJSON()` middleware formats the JSON with indentation.
   - Test asserts: status, headers preserved, JSON structure preserved, formatted output correct.

### Edge cases

- **JSON response with no `Content-Type` header:** The middleware may or may not format the response depending on implementation. This is not tested here unless regression is suspected.
- **Malformed JSON:** If the response body is not valid JSON, `c.res.clone().json()` may throw. This is existing behavior and not tested here.
- **`cancel()` throws an error:** The `Context.res` setter (Story 1) catches and suppresses the error. The formatted response is still returned. This is tested in Story 1, not here.

---

## Tests

### Summary

Add **4 regression tests** in `src/middleware/pretty-json/index.test.ts`:

1. `prettyJSON()` formats a JSON response, and the original body is cancelled (verify by mocking the body or observing the `cancel()` call)
2. `prettyJSON()` passes through a `204 No Content` response unchanged (no body cloned, no cancel invoked)
3. `prettyJSON()` passes through a non-JSON response (e.g. `text/html`) unchanged (no body cloned, no cancel invoked)
4. The formatted JSON output, status, and headers match the current behavior exactly (snapshot or explicit assertion)

**Exact test count:** 4 tests (matches breakdown.md Story 3 Verification Approach and requirements.md AC-3, AC-4)

---

## Verify

### Test coverage verification

- **AC-3 (PrettyJSON middleware cancels original body after formatting):** Covered by Test 1 (JSON formatted, original body cancelled) and Test 4 (formatted output matches current behavior)
- **AC-4 (PrettyJSON middleware preserves non-JSON and empty responses):** Covered by Test 2 (204 passthrough) and Test 3 (non-JSON passthrough)

### File path verification

- **`src/middleware/pretty-json/index.test.ts`:** Path assumed based on ticket reference. **Not confirmed in repo-tree** (repo-tree cuts off at `src/helper/cookie/index.ts`). Verify path exists in full repository before implementation. ⚠️
- **`src/middleware/pretty-json/index.ts`:** Path assumed based on middleware reference in ticket. **Not confirmed in repo-tree.** Verify path exists in full repository before implementation. ⚠️
- Both paths are concrete and follow expected Hono middleware structure

### Test count verification

- **breakdown.md:** Story 3 (PER-108) declares 4 tests
- **This LLD:** 4 tests (Test 1, Test 2, Test 3, Test 4)
- **requirements.md:** AC-3 and AC-4 both covered by the 4 tests
- Test count matches breakdown ✓

---

## Notes

- **No middleware code changes:** The cleanup is automatic via the `Context.res` setter implemented in Story 1 (PER-106). No changes to `src/middleware/pretty-json/index.ts` are required.
- **Test file creation:** `src/middleware/pretty-json/index.test.ts` does not exist and will be created in the same directory as `src/middleware/pretty-json/index.ts`. Verify the middleware directory path in the full repository before implementation.
- **Repo-tree truncation:** The provided repo-tree cuts off at `src/helper/cookie/index.ts`, so the `src/middleware/` directory is not shown. The paths `src/middleware/pretty-json/index.ts` and `src/middleware/pretty-json/index.test.ts` are assumed based on the ticket reference to the `prettyJSON()` middleware but cannot be confirmed from the truncated repo-tree. **Verify these paths exist in the full repository before implementation.**
- **Test runner:** Use Vitest and `expect` (per repo-context/hono.md: `pnpm test`)
- **Mock stream implementation:** The mock `ReadableStream` with a `cancel()` callback may need to be constructed carefully to work across all Hono runtimes (Bun, Deno, Node.js, Cloudflare Workers). If runtime-specific issues arise, adjust the mock or use a test helper.
- **Formatted JSON verification:** The tests verify formatted JSON by checking for indentation (default 2 spaces). Adjust the assertions if `prettyJSON()` uses a different indentation or formatting strategy.
- **Passthrough behavior:** Tests 2 and 3 verify that `prettyJSON()` does not transform `204` or non-JSON responses, ensuring no regression when the middleware does not apply.

---

## Dependencies

- **Story 1 (PER-106 — Implement core response body cleanup mechanism in Context):** This story depends on Story 1 being complete and all tests passing. The cleanup mechanism in `Context.res` setter must exist before these regression tests can verify it.
