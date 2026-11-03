---
key: PER-104
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: '`pnpm test` fails on `main`: published-type checks and `streamSSE` abort'
---

**Description**

`pnpm test` (`tsc -p tsconfig.spec.json && vp test --run`) exits 1 on current `main`. `pnpm lint` exits 0. These failures are already on `main` and block other work, including [PER-103](https://linear.app/personalgs/issue/PER-103/trierouter-404s-when-a-regex-parameter-is-followed-by-a-literal), because ADLC accepts a story only when `pnpm test` exits 0.

Do not change the TrieRouter or `src/utils/url.ts`. That work is [PER-103](https://linear.app/personalgs/issue/PER-103/trierouter-404s-when-a-regex-parameter-is-followed-by-a-literal).

On a checkout with no `dist/` build, these 6 tests fail:

* `runtime-tests/node/index.test.ts` — `streamSSE > Should call onAbort`. Vitest reports `Test timed out in 5000ms` at the `it('Should call onAbort')` call. The test aborts the request and then loops `while (!aborted)`.
* `build/declaration-emit.test.ts` — `published declarations > let consumers emit declarations for types inferred from hono`. `tsc` reports `TS2307: Cannot find module 'hono'`, plus `hono/factory` and `hono/testing`. The test symlinks this package as `node_modules/hono`. Package exports point at `dist/`, and `pnpm test` does not run `pnpm build`.
* `build/jsx-types.test.ts` — all 4 cases (`hono/jsx` and `hono/jsx/dom`, each with `react-jsx` and `react-jsxdev`). `tsc` reports `TS2307: Cannot find module 'hono/jsx'`. Same missing `dist/` cause. These four calls use Vitest’s default 5-second limit. `declaration-emit.test.ts` already sets `timeout: 30_000`. After `dist/` exists, these four still fail with `Test timed out in 5000ms` while `tsc` is running.

Acceptance criteria:

* `pnpm test` exits 0 on a clean checkout that has no `dist/` directory.
* `pnpm test` still exits 0 after `pnpm build`.
* `pnpm lint` still exits 0.
* The declaration and JSX tests still typecheck the published package. A missing `dist/` must not be turned into a passing assertion.
* The `streamSSE` abort test completes because `onAbort` runs, and it does not sit in an unbounded wait.
