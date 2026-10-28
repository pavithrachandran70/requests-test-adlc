# Plan — PER-69 (v1)

- made from: ticket
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: standard

Single-file behavior fix in multipart encoding that coerces non-string field names to strings, matching urlencoded behavior; needs only inline story with AC and a unit test.

## Steps

1. [ ] **analyst (inline_story)** — Ticket clearly states the bug, expected behavior, file location, and scope; one story with AC and test notes is sufficient.
   - focus: Focus on the exact coercion point in _encode_files, the parity with urlencoded behavior, and the unit test that exercises both data and files with non-string keys.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Fixed execution step.
4. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
5. [ ] **open_pr** _(auto)_ — Fixed execution step.
6. [ ] **pr_review** _(auto)_ — Fixed execution step.
7. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Scope is unambiguous: internal consistency fix with no user-facing or cross-team decisions.
- **architect** — No new components, API contracts, or cross-repo changes; single-method fix in one file.
- **scrum_master** — Work is self-contained in one repo and one story; no breakdown needed.
- **designer** — Implementation is straightforward: add str() call at field-name usage; no algorithm or file-plan detail required.
- **test_architect** — Ticket already specifies the unit test: call prepare() with non-string keys in data and files; AC maps 1:1 to test cases.
- **critic** — Only one spec document (the inline story); nothing to cross-check.
