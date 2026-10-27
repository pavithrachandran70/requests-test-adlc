```json
[]
```

NO_FINDINGS

The PR diff correctly implements all acceptance criteria:

- **AC1**: All three `iter_lines()` signatures (two overloads + implementation) now declare `chunk_size: int | None`, making the type annotation accept `None`.
- **AC2**: No runtime changes — the implementation was already passing `chunk_size` directly to `iter_content()`, which accepts `int | None`.
- **AC3**: Line-splitting behavior unchanged — only type annotations modified.
- **AC4**: No test changes required; existing tests remain valid.

The `.gitignore` changes are tooling-related and outside the scope of this type-annotation fix. The `src/requests/models.py` changes are minimal, correct, and match the technical specification exactly.
