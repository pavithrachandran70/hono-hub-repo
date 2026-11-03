# Story — PER-104-1: Fix test failures on main (missing dist/, streamSSE abort timeout)

## Metadata

```yaml
id: PER-104-1
type: story
epic: (none — quick track change)
title: Fix test failures on main (missing dist/, streamSSE abort timeout)
covers: AC-1, AC-2, AC-3
hitl: false
status: draft
assignee: (none)
tracker_id: (none)
remote: (none)
```

## Description

Fix two independent test failures blocking work on `main`:

1. **Type-check tests fail when `dist/` is missing**: `build/declaration-emit.test.ts` and the four cases in `build/jsx-types.test.ts` fail with `TS2307: Cannot find module 'hono'` because package exports point at `dist/`, and `pnpm test` does not run `pnpm build`. The JSX tests also timeout at the default 5-second limit even after `dist/` exists.

2. **`streamSSE` abort test times out**: `runtime-tests/node/index.test.ts` — `streamSSE > Should call onAbort` reports `Test timed out in 5000ms`. The test aborts the request and then loops `while (!aborted)`, but `onAbort` does not run or does not set the flag, leaving the loop unbounded.

Do not change the TrieRouter or `src/utils/url.ts` — that work is PER-103.

## Technical Specification

### Changes

#### 1. Type-check tests: handle missing `dist/`

**File: `build/declaration-emit.test.ts`**

- **Before first test**: Check if `dist/` exists with `fs.existsSync(path.join(__dirname, '../dist'))`.
- **If missing**: Run `pnpm build` with `execSync('pnpm build', { cwd: path.join(__dirname, '..'), stdio: 'inherit' })` before the test suite, or skip all tests in the file with `describe.skip` and log a clear message that published-package validation requires a build.
- **If present**: Run tests as-is.
- **Preferred approach**: Run `pnpm build` once in a `beforeAll` hook so the tests still validate the published package.

**File: `build/jsx-types.test.ts`**

- **Same check and build/skip logic** as `declaration-emit.test.ts`.
- **Set timeout**: Add `{ timeout: 30_000 }` to each of the four `it()` calls to match `declaration-emit.test.ts` and prevent timeout when `tsc` is slow.

**Imports needed** (both files):

```typescript
import fs from 'fs'
import path from 'path'
import { execSync } from 'child_process'
```

**Implementation pattern** (both files):

```typescript
import { describe, it } from 'vitest'
import fs from 'fs'
import path from 'path'
import { execSync } from 'child_process'

const distPath = path.join(__dirname, '../dist')

describe('published declarations', () => {
  if (!fs.existsSync(distPath)) {
    console.log('dist/ not found, running pnpm build...')
    execSync('pnpm build', { cwd: path.join(__dirname, '..'), stdio: 'inherit' })
  }

  it('validates published types', { timeout: 30_000 }, () => {
    // existing test logic
  })
})
```

**Alternative (skip instead of build)**: Replace the `execSync` line with `describe.skip('published declarations (skipped: dist/ missing)', () => { ... })` and wrap all tests. The ticket AC prefers the build approach to keep published-package validation.

#### 2. `streamSSE` abort test: ensure `onAbort` runs and loop terminates

**File: `runtime-tests/node/index.test.ts`**

**Current suspected issue**: The `onAbort` callback is registered but the abort signal does not fire it, or the `aborted` flag is not set, leaving `while (!aborted)` as an infinite loop.

**Fix**:

- **Verify `onAbort` is called**: Ensure the `AbortController.signal` is correctly wired to the SSE stream's abort handler. If the signal's `addEventListener('abort', ...)` is not firing, the test setup may not be triggering the abort correctly.
- **Add a timeout guard**: Even if `onAbort` should run, add a fallback to break the loop after a maximum iteration count or elapsed time so the test fails fast instead of timing out.
- **Set the flag in `onAbort`**: Confirm the callback does `aborted = true` and that the variable is not scoped incorrectly.

**Proposed fix** (assuming the flag is not being set):

Locate the `it('Should call onAbort')` test. The loop likely looks like:

```typescript
let aborted = false
const controller = new AbortController()

// register onAbort
stream.onAbort(() => {
  aborted = true
})

// abort the request
controller.abort()

// wait for onAbort
while (!aborted) {
  // spin
}
```

**Change**:

```typescript
let aborted = false
const controller = new AbortController()

stream.onAbort(() => {
  aborted = true
})

controller.abort()

// Wait for onAbort with a timeout
const start = Date.now()
while (!aborted) {
  if (Date.now() - start > 1000) {
    throw new Error('onAbort did not run within 1 second')
  }
  await new Promise(resolve => setTimeout(resolve, 10))
}

// Assert onAbort was called
expect(aborted).toBe(true)
```

**Alternative fix (if the abort signal is not wired)**:

If the SSE stream does not listen to `controller.signal`, wire it:

```typescript
controller.signal.addEventListener('abort', () => {
  stream.onAbort?.()
})
```

Then `controller.abort()` will fire the listener, which calls the registered `onAbort` callback.

**Note**: The exact fix depends on the SSE implementation in the codebase. Inspect `runtime-tests/node/index.test.ts` and the SSE stream class to confirm how `onAbort` is wired to the abort signal. The timeout guard is the safest immediate fix.

### File Tree

```
build/
  declaration-emit.test.ts  (add dist/ check and build/skip logic)
  jsx-types.test.ts         (add dist/ check, build/skip, and timeout: 30_000)
runtime-tests/
  node/
    index.test.ts           (fix onAbort callback and loop timeout)
```

No new files. All changes are in existing test files.

## Acceptance Criteria

### AC-1: Type-check tests handle missing dist/

**Given** a clean checkout with no `dist/` directory  
**When** `pnpm test` runs  
**Then** the declaration-emit and jsx-types tests either build `dist/` first or skip gracefully with a clear message, and `pnpm test` exits 0

**Given** `dist/` exists from a prior build  
**When** `pnpm test` runs  
**Then** the declaration-emit and jsx-types tests validate the published package as they do now, and `pnpm test` exits 0

**Given** the declaration-emit and jsx-types tests detect missing `dist/`  
**When** they decide to skip  
**Then** they do not report the skip as a passing assertion — the test outcome reflects that published-package validation did not run

### AC-2: streamSSE abort test completes without timeout

**Given** the `streamSSE > Should call onAbort` test runs  
**When** the request is aborted  
**Then** `onAbort` executes, sets the `aborted` flag, and the `while (!aborted)` loop terminates within 5 seconds

**Given** `onAbort` is registered in the test  
**When** the abort signal fires  
**Then** the callback runs and updates state so the loop exits

### AC-3: No regression in passing tests or lint

**Given** `pnpm test` and `pnpm lint` both exit 0 on current `main` after `pnpm build`  
**When** the fixes land  
**Then** both still exit 0 after the build

**Given** the fixes change test setup or assertions  
**When** other tests run  
**Then** no previously passing test fails

## Verify

- Run `pnpm test` on a clean checkout (no `dist/`) and confirm exit 0, with either dist/ built or tests skipped
- Run `pnpm build && pnpm test` and confirm exit 0 with all type-check tests passing
- Run `pnpm test` and confirm the `streamSSE > Should call onAbort` test completes in under 5 seconds
- Run `pnpm lint` and confirm exit 0

## Notes

- The two fixes are independent and can be tested separately.
- The `declaration-emit.test.ts` already sets `timeout: 30_000`; the JSX tests need the same to avoid timeout during slow `tsc` runs.
- The exact `streamSSE` abort fix depends on the SSE implementation — inspect `runtime-tests/node/index.test.ts` and the stream class to confirm how `onAbort` is wired.
- The ticket explicitly excludes TrieRouter or `src/utils/url.ts` changes — that work is PER-103.
