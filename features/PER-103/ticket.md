---
key: PER-103
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: "TrieRouter 404s when a regex parameter is followed by a literal\_ `[segment]`"
---

Description

`GET /a/42/[draft]` is registered as `/a/:id{[0-9]+}/[draft]`. `[draft]` is a literal path segment. RegExpRouter returns `200` with body `42`. TrieRouter returns `404`. The default app uses SmartRouter. With only that route it stays on RegExpRouter and returns `200`, but adding unrelated routes makes SmartRouter select TrieRouter and the same request returns `404`.

Confirmed on this checkout:

```
TrieRouter TrieRouter 404 404 Not Found
RegExpRouter RegExpRouter 200 42
default SmartRouter + RegExpRouter 200 42
default with extra routes SmartRouter + TrieRouter 404 404 Not Found
```

`getPattern()` in `src/utils/url.ts` puts the next segment into a regexp lookahead, so `[draft]` is read as a character class. `src/router/linear-router/router.ts` builds that lookahead separately. A following static segment such as `file.html` already works and must keep working.

Acceptance criteria:

* TrieRouter, RegExpRouter, LinearRouter, and the default SmartRouter all return `200` and body `42` for `GET /a/42/[draft]`.
* The SmartRouter case still returns `200` after unrelated routes are added and it selects TrieRouter.
* `GET /a/42/draft` and `GET /a/42/[other]` stay unmatched.
* An existing route whose next segment is `file.html` still matches.
* Regression tests cover TrieRouter and the SmartRouter fallback. Run the other router suites too.
* `pnpm test` and `pnpm lint` pass.
* Do not change route syntax or the public `Hono` constructor. Open the pull request on `pavithrachandran70/hono`, not `honojs/hono`.
