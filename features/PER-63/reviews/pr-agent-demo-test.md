```json
[
  {
    "path": ".gitignore",
    "line": 38,
    "body": "**AC gap:** The PR only adds `.gitignore` entries for ADLC/Cursor tooling. No changes to `src/requests/adapters.py` are present — the two `OSError` raises for missing cert and key paths remain unchanged.\n\n**Expected:** Changes to `HTTPAdapter.cert_verify` in `src/requests/adapters.py` replacing `raise OSError(...)` with `raise FileNotFoundError(errno.ENOENT, ..., <path>)` for both cert and key missing-path checks.\n\n**Consequence:** AC1, AC2, AC4, and AC5 cannot pass — the exception type remains `OSError`, no `filename` attribute is set, and no new tests exist."
  },
  {
    "path": "N/A",
    "line": null,
    "body": "**AC gap:** No new tests in `tests/test_adapters.py` (or any test file). AC4 requires unit tests for missing cert path → `FileNotFoundError` with `filename` attribute, and missing key path → `FileNotFoundError` with `filename` attribute.\n\n**Expected per LLD:** New tests in `tests/test_adapters.py`: `test_cert_verify_missing_cert_path_raises_file_not_found_error`, `test_cert_verify_missing_key_path_raises_file_not_found_error`, `test_cert_verify_file_not_found_error_has_filename_attribute`, `test_cert_verify_file_not_found_error_caught_by_oserror`.\n\n**Consequence:** AC1, AC2, AC4 cannot be verified — no test coverage for the new exception contract."
  },
  {
    "path": ".gitignore",
    "line": 38,
    "body": "**Wrong scope:** `.gitignore` changes are unrelated to PER-63's acceptance criteria. The story scope is `src/requests/adapters.py` (two exception raises) and `tests/test_adapters.py` (new tests). Tooling ignore patterns are out of scope.\n\n**Consequence:** The PR addresses a different concern (dev environment hygiene) and does not implement the feature."
  }
]
```
