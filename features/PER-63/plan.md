# Plan — PER-63 (v1)

- made from: ticket
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: standard

A focused exception-type change in one method with clear acceptance criteria; analyst inline_story provides the implementable story and test_architect adds test cases.

## Steps

1. [ ] **analyst (inline_story)** — Ticket scope is concrete and self-contained: change two raises and add a test.
   - focus: Keep the story to the exact scope: update two raises in cert_verify, add one unit test, update any tests that assert the exact exception type.
2. [ ] **test_architect (tests)** — Acceptance criteria specify multiple edge cases (missing cert, missing key, filename attribute, backward compatibility with OSError catch).
   - focus: Map each AC bullet to a test case: missing cert path, missing key path, filename attribute presence, existing tests still pass.
3. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
4. [ ] **dev** _(auto)_ — Fixed execution step.
5. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
6. [ ] **open_pr** _(auto)_ — Fixed execution step.
7. [ ] **pr_review** _(auto)_ — Fixed execution step.
8. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Ticket scope and acceptance criteria are unambiguous and purely technical.
- **architect** — No API contract, schema, or cross-repo changes; the change is localized to exception handling.
- **scrum_master** — Single story in one repo; no breakdown needed.
- **designer** — Implementation is straightforward: replace OSError with FileNotFoundError in two places.
- **critic** — Only one spec artifact (the story with tests); nothing to cross-check.
