# Plan — PER-71 (v2)

- made from: review feedback (architect_review)
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: quick

Single-repo bug fix in super_len for StringIO UTF-8 byte length; architect added per reviewer to write HLD constraining the change site before Dev.

## What changed

Added architect role to write HLD constraining the fix to super_len only; analyst reruns to update requirements and story with the design constraint.

- Added: architect
- Rerun: analyst

## Feedback that informed this version

- **PR comment** by pavithrachandran70 on hub#11 `[hub#11:issue-6465451991]`: The plan skips the architect, but the fix location is a design decision. Add the architect role and write an HLD before Dev.

Design constraint: change only super_len in src/requests/utils.py. When the stream is a text file object (io.TextIOBase, including StringIO), Content-Length must be the UTF-8 byte length, not the character count. Do not change prepare_body in src/requests/models.py.

Binary BytesIO must keep its current byte length. An explicit Content-Length from the caller must not be overwritten. Multipart and JSON bodies stay unchanged.

Update requirements.md and stories/01-demo-test.md so the change site is super_len only, and add an acceptance criterion: StringIO("héllo") prepares with Content-Length 6, while BytesIO("héllo".encode()) stays Content-Length 6.
  - addressed by: Added architect role to produce HLD; analyst reruns with instructions to update requirements.md and stories/01-demo-test.md so change site is super_len only, and add AC for StringIO('héllo') and BytesIO('héllo'.encode()).

## Steps

1. [x] **analyst (inline_story)** — Single-repo behavioral fix with clear acceptance criteria stated in the ticket.
   - focus: Update requirements.md and stories/01-demo-test.md so the change site is super_len only in src/requests/utils.py. Add AC: StringIO('héllo') prepares with Content-Length 6, BytesIO('héllo'.encode()) stays 6. Do not change prepare_body in src/requests/models.py.
2. [x] **architect (hld)** — Reviewer feedback: fix location is a design decision; HLD must constrain the change to super_len only.
   - focus: Write HLD constraining the fix to super_len in src/requests/utils.py. When the stream is io.TextIOBase (including StringIO), Content-Length must be UTF-8 byte length, not character count. Do not change prepare_body in src/requests/models.py. Binary BytesIO keeps current byte length. Explicit Content-Length from caller must not be overwritten. Multipart and JSON bodies stay unchanged.
3. [x] **merge_pack (pack)** _(auto)_ — Pack per-ticket specs into merged_plan.md for review.
4. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Fixed execution step.
6. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
7. [ ] **open_pr** _(auto)_ — Fixed execution step.
8. [ ] **pr_review** _(auto)_ — Fixed execution step.
9. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Scope is clear and technical; no product sign-off needed for a bug fix.
- **scrum_master** — Work does not split into multiple stories or repos.
- **designer** — HLD provides sufficient implementation constraint; no separate file plan needed.
- **test_architect** — AC already maps 1:1 to test cases; no additional test design required.
- **critic** — Two documents (requirements, HLD) both trace to the same ticket; consistency is implicit.
