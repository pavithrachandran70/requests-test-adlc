```json
[
  {
    "lens": "adversarial",
    "location": "tests/test_requests.py:980-984",
    "trigger_condition": "Test expects `IOError` but catches `FileNotFoundError` — `IOError` is an alias for `OSError`, not the parent of `FileNotFoundError`",
    "guard_snippet": "Change `pytest.raises(FileNotFoundError)` to `pytest.raises(OSError)` to match the original intent (catch the hierarchy root), or keep `FileNotFoundError` and document the intentional narrowing",
    "potential_consequence": "Test now catches only FileNotFoundError; a future refactor that raises a different OSError subclass (e.g. PermissionError for unreadable files) will fail this test even though the original `IOError` catch would have passed"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_requests.py:986-991",
    "trigger_condition": "Same issue: test expects `IOError` but catches `FileNotFoundError`",
    "guard_snippet": "Change `pytest.raises(FileNotFoundError)` to `pytest.raises(OSError)` or document the narrowing",
    "potential_consequence": "Test fragility — a different OSError subclass will break the test"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:10-11",
    "trigger_condition": "Class `_Conn` has no docstring explaining what urllib3 contract it satisfies or why it's a bare object",
    "guard_snippet": "Expand docstring: 'Minimal stand-in for urllib3.HTTPSConnection. cert_verify only reads/writes cert_file and key_file attributes; no methods required.'",
    "potential_consequence": "Next maintainer doesn't know if this mock is sound or which urllib3 type it replaces"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:21-35",
    "trigger_condition": "Test creates a missing path `/tmp/does-not-exist-adlc-cert.pem` without cleanup, and depends on it not existing — a parallel test run or leftover file breaks the assertion",
    "guard_snippet": "Use `tmp_path` fixture: `missing_cert_path = tmp_path / 'cert.pem'; assert not missing_cert_path.exists()`",
    "potential_consequence": "Flaky test when `/tmp/does-not-exist-adlc-cert.pem` exists from a prior run"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:38-60",
    "trigger_condition": "Test creates a temp file with `delete=False` and then `os.unlink` in finally — if the test crashes before the unlink or the path is wrong, the file leaks",
    "guard_snippet": "Use `tmp_path` fixture: `cert_path = tmp_path / 'cert.pem'; cert_path.write_text('dummy'); missing_key = tmp_path / 'key.pem'` — no manual cleanup needed",
    "potential_consequence": "Leaked temp files on test failure or crash"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:63-78",
    "trigger_condition": "Test uses bare `except Exception: pass` after catching `OSError` — swallows assertion failures and makes the test pass even when the code raises something completely different",
    "guard_snippet": "Remove the `except Exception: pass` clause entirely; let the test fail if a non-OSError is raised, or replace with `except Exception as unexpected: pytest.fail(f'Raised {type(unexpected).__name__} instead of OSError')`",
    "potential_consequence": "Test silently passes when the code raises TypeError, AttributeError, or any other exception; the assertion `caught_as_oserror` is False and the test claims to pass but actually failed"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:21-35, 38-60, 63-78",
    "trigger_condition": "No test verifies the `errno.ENOENT` value when catching the exception as `OSError` (only as `FileNotFoundError` in the first two tests)",
    "guard_snippet": "In `test_file_not_found_error_is_caught_by_oserror`, add `assert exc.errno == errno.ENOENT` after catching as OSError to confirm the errno survives the hierarchy",
    "potential_consequence": "AC1/AC2 claim the errno is set, but no test confirms it's accessible through the OSError catch path"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:21-78",
    "trigger_condition": "No test for AC3 'existing tests remain compatible' — the diff updates `test_requests.py` to expect `FileNotFoundError`, but no test verifies that old code expecting `OSError` (not `IOError`) still works",
    "guard_snippet": "Add a test that catches `OSError` generically and asserts `isinstance(exc, FileNotFoundError)` to prove the subclass relationship",
    "potential_consequence": "AC3 is not verified by any new test — the only 'proof' is the manual change to existing tests"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:21-78",
    "trigger_condition": "AC4 says 'all tests pass, including new tests for missing cert and missing key paths' — but the diff removes two tests (`test_custom_sslcontext_preserves_verify_mode` and `test_default_verify_enforces_cert_required`) without explanation",
    "guard_snippet": "Restore the removed tests or document why they're deleted (e.g. 'obsolete after urllib3 2.0 refactor') in the commit message",
    "potential_consequence": "Coverage loss — those tests verified SSL context handling; deleting them is out of scope for this story"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:21-78",
    "trigger_condition": "AC5 'no behavior change when files exist' is not tested — no test creates valid cert and key files and confirms the adapter proceeds without raising",
    "guard_snippet": "Add a test: create two temp files with dummy content, pass them as `cert=(cert_path, key_path)`, and assert `cert_verify` completes without exception",
    "potential_consequence": "AC5 is not verified; a regression in the file-exists path would not be caught"
  },
  {
    "lens": "adversarial",
    "location": "src/requests/adapters.py:367-370",
    "trigger_condition": "The new `FileNotFoundError` constructor takes three positional args `(errno, strerror, filename)` — but Python's FileNotFoundError can also take two args `(errno, strerror)` and the filename as a kwarg, and the diff does not show which form is canonical in this codebase",
    "guard_snippet": "Confirm the three-arg form matches urllib3's or requests' existing exception patterns; if not, use `FileNotFoundError(errno.ENOENT, 'message', filename=conn.cert_file)` for clarity",
    "potential_consequence": "Inconsistent exception construction if other code uses the two-arg form or kw-only filename"
  },
  {
    "lens": "adversarial",
    "location": "src/requests/adapters.py:367-370, 372-377",
    "trigger_condition": "The message string changes from f-string `f'...invalid path: {conn.cert_file}'` to a plain string `'...invalid path'` with filename as the third arg — the rendered message format is now `[Errno 2] Could not find the TLS certificate file, invalid path: /path/to/file` instead of the old `Could not find the TLS certificate file, invalid path: /path/to/file`",
    "guard_snippet": "Verify the message format change is acceptable; if not, include the path in the message string: `f'Could not find the TLS certificate file, invalid path: {conn.cert_file}'` and drop the filename arg, or adjust tests to match the new format",
    "potential_consequence": "Message format breaks scripts or logs that parse the old format"
  }
]
```
