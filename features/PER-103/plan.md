# Plan — PER-103 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Bug fix in TrieRouter's regex parameter handling when followed by a literal segment — quick_fix fits because the behavior is stated, the fix is in one repo, and no product sign-off or breakdown is needed.

## Steps

1. [ ] **analyst (inline_story)** — Turn the ticket into a single implementable story with Given/When/Then AC and technical notes on the getPattern() lookahead bug.
   - focus: Focus on the getPattern() character class interpretation bug. Include technical notes on how LinearRouter already handles this correctly and the file plan showing which router files need changes. Map the multiple router test cases (TrieRouter, RegExpRouter, SmartRouter combinations, negative cases, file.html preservation) directly into the AC section so they are clear for implementation.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Only one story and no multi-document consistency to check.

## Planner notes

- attempt 1: designer is not a step of loop quick_fix; test_architect is not a step of loop quick_fix
