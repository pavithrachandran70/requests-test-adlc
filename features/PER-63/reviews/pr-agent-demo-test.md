```json
[
  {
    "path": "src/requests/adapters.py",
    "line": 367,
    "body": "**AC1 violation:** Missing cert path raises `FileNotFoundError` but the message format is broken. The three-argument form `FileNotFoundError(errno, msg, filename)` produces `[Errno 2] Could not find the TLS certificate file, invalid path: '/path/to/cert'` — the filename is appended automatically with a colon separator, so the message text should **not** include \"invalid path\" as a suffix. Current output: `[Errno 2] Could not find the TLS certificate file, invalid path: /tmp/cert.pem` (redundant \"invalid path\"). Either use the two-argument form `FileNotFoundError(msg.format(cert_file))` and set `exc.filename = cert_file` afterward, or change the message to end before the filename (e.g. `\"Could not find the TLS certificate file\"`). The same issue exists at line 373 for the key path."
  },
  {
    "path": "src/requests/adapters.py",
    "line": 367,
    "body": "**Inconsistency:** The cert-path raise uses `\"Could not find the TLS certificate file, invalid path\"` (no comma before the filename auto-append), but line 373 for the key path uses `\"Could not find the TLS certificate key file, invalid path\"` — both carry the same redundant suffix. The shipped message format was `f\"Could not find the TLS certificate file, invalid path: {conn.cert_file}\"` — preserve that exact text by using a single-argument raise with the formatted string, then set `.filename` afterward: `exc = FileNotFoundError(f\"Could not find the TLS certificate file, invalid path: {conn.cert_file}\"); exc.filename = conn.cert_file; raise exc`. Or drop \"invalid path\" from the message entirely and rely on the auto-appended filename."
  },
  {
    "path": "tests/test_adapters.py",
    "line": 52,
    "body": "**AC3/AC4 gap:** Test coverage for \"existing tests remain compatible\" is incomplete. The diff removes two tests (`test_custom_sslcontext_preserves_verify_mode` and `test_default_verify_enforces_cert_required`) that are **unrelated** to cert/key path validation — they test pool manager SSL context behavior, not missing file raises. Removing them violates AC3 (\"existing tests still pass **or** are updated only where they assert the exact exception type or message text\") unless the removal is explained as cleanup outside this story's scope. If these tests failed or became obsolete, document why; otherwise restore them."
  },
  {
    "path": "tests/test_requests.py",
    "line": 979,
    "body": "**AC1/AC2 verification strengthened:** The updated test at line 979 now asserts `isinstance(e.value, OSError)` and checks the `filename` attribute — good. However, it also checks that the message contains `\"Could not find the TLS certificate file, invalid path\"` (line 983), which will **fail** with the current three-argument raise form because the auto-generated message is `[Errno 2] Could not find the TLS certificate file, invalid path: /garbage` (note the `[Errno 2]` prefix and the colon before the filename). The test expects the substring without the errno prefix, so it will pass only if the message format is adjusted or the test expectation accounts for the prefix."
  },
  {
    "path": "tests/test_adapters.py",
    "line": 21,
    "body": "**AC5 gap:** No test for \"no behavior change when files exist.\" The new tests cover missing cert and missing key paths, but none verify that when `cert` points to a valid file (or a valid `(cert, key)` tuple), `cert_verify` does not raise and sets `conn.cert_file` and `conn.key_file` correctly. Add a test: create two real temp files, pass them as `cert=(cert_path, key_path)`, assert no exception and `conn.cert_file == cert_path`, `conn.key_file == key_path`."
  },
  {
    "path": "tests/test_adapters.py",
    "line": 30,
    "body": "**Test hygiene:** `test_missing_key_raises_file_not_found_error` creates a temp file `valid_cert_path` but does not verify that it actually exists before running the test. If `tempfile.NamedTemporaryFile` fails or the file is immediately deleted (depending on OS), the test could spuriously pass (because the cert path is also missing, triggering the cert-missing raise before the key-missing raise). Add `assert os.path.exists(valid_cert_path)` after the file is written and before invoking `cert_verify`."
  },
  {
    "path": "tests/test_adapters.py",
    "line": 63,
    "body": "**Test clarity:** `test_file_not_found_error_is_caught_by_oserror` sets `caught_as_oserror = False`, then flips it to `True` in the `except OSError` block — but the test does not assert that the exception was actually raised. If `cert_verify` silently succeeds (a regression), `caught_as_oserror` remains `False` and the assertion fails with a misleading message. Instead, use `pytest.raises(FileNotFoundError)` and then verify `isinstance(exc_info.value, OSError)` — clearer and catches the \"no exception raised\" case."
  },
  {
    "path": "tests/test_requests.py",
    "line": 986,
    "body": "**AC2 verification:** The second half of the test (line 986–991) checks the missing key path case, but the cert argument is `('.')` — the current directory. On most systems `.` exists, so the cert-path check passes and the key-path check is reached. However, if the test runs in an environment where `.` does not exist (e.g. a restricted container), the test will fail on the cert-path check instead of the key-path check. Use a known-valid cert path (create a temp file, like in `test_adapters.py::test_missing_key_raises_file_not_found_error`) or assert that `.` exists before running this half of the test."
  }
]
```
