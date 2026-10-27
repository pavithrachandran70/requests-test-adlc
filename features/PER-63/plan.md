# Plan — PER-63 (v3)

- made from: review feedback (plan_review)
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: standard

Single-repo exception contract change: replace two OSError raises with FileNotFoundError in cert_verify, set filename attribute, preserve OSError compatibility, and add tests for edge cases.

## What changed

Added designer step to specify the exact FileNotFoundError raises for missing cert and key paths, including filename attribute handling and CA-bundle preservation.

- Added: designer

## Feedback that informed this version

- **PR comment** by pavithrachandran70 on hub#4 `[hub#4:issue-6440350463]`: 2. Add designer

This is a caller-visible exception contract, not a class-name swap. Add a designer step that names the exact raise for the missing cert path and the missing key path, including how .filename is set, and that says the CA-bundle raise is unchanged. The skip reason "no API contract" does not match the ticket.
  - addressed by: Added designer step with focus on naming exact raise statements for missing cert/key paths, filename parameter, and unchanged CA-bundle raise.

## Steps

1. [x] **analyst (inline_story)** — Ticket scope is concrete: change two specific raises (cert and key), preserve the third (CA-bundle), set filename attribute, and maintain OSError compatibility.
2. [x] **designer (lld)** — Caller-visible exception contract: must specify exact raise statements for missing cert and key paths, how filename is set, and that CA-bundle raise is unchanged.
   - focus: Name the exact FileNotFoundError raise for missing cert path and missing key path, including how the filename parameter is set. Confirm the CA-bundle raise is unchanged. This is an exception contract, not an implementation detail.
3. [x] **test_architect (tests)** — Acceptance criteria specify multiple edge cases: missing cert, missing key, filename attribute, backward compatibility with OSError catch, and message format for existing tests.
4. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Fixed execution step.
6. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
7. [ ] **open_pr** _(auto)_ — Fixed execution step.
8. [ ] **pr_review** _(auto)_ — Fixed execution step.
9. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Ticket scope and acceptance criteria are unambiguous and purely technical.
- **architect** — No new components, cross-repo dependencies, or schema changes; the exception contract is handled by the designer.
- **scrum_master** — Single story in one repo; no breakdown needed.
- **critic** — Only one spec artifact (the story with tests); nothing to cross-check.
