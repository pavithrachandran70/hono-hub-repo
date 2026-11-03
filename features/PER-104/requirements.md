# requirements.md

## Overview

Fix two independent test failures on `main`: (1) type-check tests that fail when `dist/` is missing, and (2) the `streamSSE` abort test that times out.

## Given/When/Then Acceptance Criteria

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

## Open Questions

- None — the ticket supplies the failure symptoms and acceptance criteria.

## Notes

- The ticket explicitly states "Do not change the TrieRouter or `src/utils/url.ts`" — that work is PER-103.
- The two fixes are independent: missing `dist/` affects type-check tests, and the abort timeout is a separate runtime-test issue.
- The declaration-emit test already sets `timeout: 30_000`. The four jsx-types cases need the same or must skip when `dist/` is missing.
