**Wrong scope:** `.gitignore` changes are unrelated to PER-63's acceptance criteria. The story scope is `src/requests/adapters.py` (two exception raises) and `tests/test_adapters.py` (new tests). Tooling ignore patterns are out of scope.

**Consequence:** The PR addresses a different concern (dev environment hygiene) and does not implement the feature.
