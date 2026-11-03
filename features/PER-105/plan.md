# Plan — PER-105 (v2)

- made from: review feedback (plan_review)
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

This feature fixes resource cleanup when middleware replaces c.res, affecting multiple middleware components and the response contract, requiring architecture and design work.

## What changed

Switched from quick_fix to feature_with_architecture, adding architect, scrum_master, designer, and critic to handle the cross-middleware response contract change.

- Added: architect, scrum_master, designer, critic

## Feedback that informed this version

- **PR comment** by pavithrachandran70 on hub#5 `[hub#5:issue-6601858716]`: Change the loop from quick_fix to feature_with_architecture.

Keep these steps, in this order:
- analyst with activity requirements, not inline_story
- architect, so it writes hld.md and the specs
- scrum_master, so it writes breakdown.md and the stories
- designer, so each story gets a low-level design
- critic

This change crosses middleware and the response contract, so one inline story is not enough. Do not skip the architect or the designer.
  - addressed by: Changed loop to feature_with_architecture; added analyst (requirements), architect, scrum_master, designer, and critic as requested.

## Steps

1. [x] **analyst (requirements)** — Extract structured requirements from the ticket's bug description and multi-middleware acceptance criteria.
   - focus: Focus on the resource cleanup contract when c.res is replaced, covering all affected middleware (etag, prettyJSON, cache, compress, timeout, jsx-renderer, secure-headers).
2. [x] **architect (hld)** — Design the response body cleanup contract and the mechanism for releasing discarded bodies across middleware.
   - focus: Define the contract for cleanup when c.res is replaced, ensuring stream cancellation and clone() cleanup work consistently across all middleware without breaking existing behavior.
3. [x] **scrum_master (breakdown)** — Break down the work across affected middleware and the core response-replacement mechanism.
   - focus: Create stories for the core cleanup mechanism, etag fixes, prettyJSON fixes, verification of other middleware, and regression tests.
4. [x] **designer (lld)** — Each story needs concrete implementation detail for the cleanup logic and test cases.
   - focus: Provide file plans and implementation notes for the response body cleanup mechanism, ensuring error handling doesn't break request processing.
5. [ ] **critic (consistency_check)** — Ensure consistency between HLD contract design, breakdown across middleware, and low-level implementation details.
   - focus: Verify the cleanup contract in the HLD matches the implementation approach in each story, and test coverage aligns with all acceptance criteria.
6. [ ] **merge_pack (pack)** _(auto)_ — Pack per-ticket specs into merged_plan.md for review.
7. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
8. [ ] **dev** _(auto)_ — Loop delivery.
9. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
10. [ ] **open_pr** _(auto)_ — Loop delivery.
11. [ ] **pr_review** _(auto)_ — Loop delivery.
12. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clearly stated in the ticket; this is a technical bug fix with defined behavior.
- **test_architect** — Test cases map directly to the seven acceptance criteria and are straightforward enough to be handled in the designer's test sections.
