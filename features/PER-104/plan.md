# Plan — PER-104 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Fix test failures on main caused by missing dist/ in type-check tests and a timeout in the streamSSE abort test; quick_fix fits because the behavior is clear and the fix is single-repo.

## Steps

1. [x] **analyst (inline_story)** — The ticket clearly states the problem and acceptance criteria; one story captures the fix.
   - focus: Focus on two independent fixes: (1) make type-check tests run pnpm build first or skip when dist/ is missing, keeping the published-package validation; (2) fix the streamSSE abort test timeout by ensuring onAbort runs and the loop terminates.
2. [x] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [x] **dev** _(auto)_ — Loop delivery.
4. [x] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [x] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Only one inline story is produced, so there are no multiple artifacts to cross-check for consistency.
