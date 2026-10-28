# Plan — PER-71 (v1)

- made from: ticket
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: quick

A focused bug fix in one repo's length calculation for text streams; analyst inline_story alone produces the implementable story with AC and tech notes.

## Steps

1. [x] **analyst (inline_story)** — Single-repo behavioral fix with clear acceptance criteria stated in the ticket.
   - focus: Focus on the UTF-8 encoding edge case in super_len or prepare_body; note that binary streams must not change.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Fixed execution step.
4. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
5. [ ] **open_pr** _(auto)_ — Fixed execution step.
6. [ ] **pr_review** _(auto)_ — Fixed execution step.
7. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Scope is clear and technical; no product sign-off needed for a bug fix.
- **architect** — No new components, API contracts, or cross-repo changes; fix is local to length calculation.
- **scrum_master** — Work does not split into multiple stories or repos.
- **designer** — Implementation path is straightforward; no file plan needed before coding.
- **test_architect** — AC already maps 1:1 to test cases; no additional test design required.
- **critic** — Single document output; nothing to cross-check for consistency.
