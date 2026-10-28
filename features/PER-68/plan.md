# Plan — PER-68 (v1)

- made from: ticket
- source: LLM planner
- complexity: moderate
- needs plan review: yes
- track hint: standard

Digest authentication rebuild for correct URI handling with semicolons, fragments, and redirects requires inline story with implementation detail, plus test coverage to verify edge cases.

## Steps

1. [ ] **analyst (inline_story)** — Clear behavioral requirements with specific URL cases and AC map directly to one implementable story.
   - focus: Focus on the four URL cases, the MD5 response change when semicolons are included, and the redirect behavior differences between same-host 307 and cross-host redirects.
2. [ ] **designer (lld)** — Digest auth logic and redirect handling involve precise algorithm changes across two files that need a concrete implementation plan.
   - focus: Detail the URI extraction logic to preserve semicolons and strip fragments, the digest rebuild trigger in redirect paths, and how to verify the response hash changes.
3. [ ] **test_architect (tests)** — Four specific URL cases plus redirect scenarios and hash verification are edge cases worth pinning as explicit test cases.
   - focus: Map each of the four URL cases to a unit test, add tests for the MD5 response hash change, and cover both same-host 307 and cross-host redirect behaviors.
4. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Fixed execution step.
6. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
7. [ ] **open_pr** _(auto)_ — Fixed execution step.
8. [ ] **pr_review** _(auto)_ — Fixed execution step.
9. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **architect** — No new components, API contracts, or cross-repo changes; work is scoped to existing auth and session modules.
- **scrum_master** — Single repo, single story with clear scope; no breakdown needed.
- **tpm_gate** — Requirements are technical and unambiguous with explicit acceptance criteria; no product scope ambiguity.
- **critic** — Single story produces one set of specs; no multiple artifacts to cross-check for consistency.
