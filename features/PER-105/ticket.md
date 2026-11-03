---
key: PER-105
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: "Release the discarded response body when middleware replaces\_ `c.res`"
---

When middleware replaces `c.res` with a new `Response`, the previous response body is dropped and never cancelled. Stream `cancel()` callbacks do not run, and the unread half of a `clone()` stays buffered.

This shows up in two places on `main`:

* `etag()` returns `304 Not Modified` and replaces `c.res`. The handler stream's `cancel()` is never called. The same happens when the `ETag` is computed from `res.clone().body`: the original branch stays open.
* `prettyJSON()` reads `c.res.clone().json()`, then replaces `c.res` with the formatted body. A reader on the original body still reports `done: false`.

Other middleware also replace `c.res`, including `cache`, `compress`, `timeout`, `jsx-renderer`, and `secure-headers`. The visible behavior of each one must stay the same.

Acceptance criteria:

* `etag()` returns `304` with the same status, `ETag`, and retained headers as today, and the discarded body is cancelled. This covers both a preset `ETag` and an `ETag` generated from the body.
* `prettyJSON()` returns the same formatted JSON, status, and headers as today, and the original body is cancelled. An empty `204` and a non-JSON body still pass through unchanged.
* Replacing `c.res` with a response that keeps the previous body still streams that body to the end.
* A failure while releasing the old body does not turn the request into a `500`. The new response is still returned.
* `c.res` keeps its current header merging, including `set-cookie`.
* Regression tests cover both `etag` cases, `prettyJSON`, a reused body, and a failed release.
* `pnpm test` and `pnpm lint` pass.
* Do not change public middleware options or `Context` method signatures. Open the pull request on `pavithrachandran70/hono`, not `honojs/hono`.
