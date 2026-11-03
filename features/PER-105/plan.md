# Plan — PER-105 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

A bug fix in one repo (hono) that updates middleware to cancel discarded response bodies when c.res is replaced; the ticket states clear behavior and acceptance criteria, so quick_fix is appropriate.

## Steps

1. [ ] **analyst (inline_story)** — The ticket describes the bug and acceptance criteria clearly but needs a structured story with implementation notes.
   - focus: Write a complete implementable story with file plan, cancellation logic, and test cases. Focus on etag() and prettyJSON() middleware as primary examples; note that other middleware (cache, compress, timeout, jsx-renderer, secure-headers) must retain their current behavior. Include concrete implementation guidance on where to cancel bodies, how to handle clone() branches, error handling that prevents 500 responses, and regression test cases for: etag with preset ETag, etag with generated ETag, prettyJSON formatting, prettyJSON passthrough (204, non-JSON), body reuse, failed release, and header merging with set-cookie.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Only one spec artifact (the inline story) exists, so there is nothing to cross-check for consistency.

## Planner notes

- attempt 1: designer is not a step of loop quick_fix; test_architect is not a step of loop quick_fix
