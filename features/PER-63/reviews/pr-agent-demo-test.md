```json
[
  {
    "path": "src/requests/adapters.py",
    "line": 367,
    "body": "**AC1 verified:** Missing cert path now raises `FileNotFoundError` with `errno.ENOENT` and the missing path as `filename`. The three-argument constructor is correct: `(errno, message, filename)`. `FileNotFoundError` is a subclass of `OSError`, so existing `except OSError` blocks remain compatible."
  },
  {
    "path": "src/requests/adapters.py",
    "line": 372,
    "body": "**AC2 verified:** Missing key path now raises `FileNotFoundError` with `errno.ENOENT` and the missing key path as `filename`. The constructor matches the cert path raise. Backward compatibility preserved."
  },
  {
    "path": "tests/test_adapters.py",
    "line": 21,
    "body": "**AC1 test added:** `test_missing_cert_raises_file_not_found_error` verifies the exception type, subclass relationship, errno, filename attribute, and message content. Covers AC1 completely."
  },
  {
    "path": "tests/test_adapters.py",
    "line": 37,
    "body": "**AC2 test added:** `test_missing_key_raises_file_not_found_error` creates a temporary valid cert file, passes a missing key path, and verifies the exception type, subclass relationship, errno, filename attribute, and message content. Covers AC2 completely. Cleanup via `os.unlink` ensures no test artifacts remain."
  },
  {
    "path": "tests/test_adapters.py",
    "line": 66,
    "body": "**AC1/AC2 backward-compatibility test added:** `test_file_not_found_error_is_caught_by_oserror` confirms that `except OSError` catches the new `FileNotFoundError`. Demonstrates that existing error-handling code remains valid."
  },
  {
    "path": "tests/test_requests.py",
    "line": 979,
    "body": "**AC3 verified:** Existing test updated from `IOError` (alias for `OSError`) to `FileNotFoundError`. The test now asserts `isinstance(e.value, OSError)` to confirm subclass compatibility, checks `e.value.filename`, and verifies the message contains the expected text. No behavior regression — the test still validates the same contract."
  },
  {
    "path": "tests/test_requests.py",
    "line": 986,
    "body": "**AC3 verified:** Existing test for missing key path updated to `FileNotFoundError`, with `isinstance(OSError)` check, `filename` assertion, and message validation. Same scope as the cert-path update — backward compatibility preserved, no new behavior introduced."
  },
  {
    "path": "tests/test_adapters.py",
    "line": 21,
    "body": "**AC4 implicit:** The new tests are TLS-cert-related unit tests. They verify missing cert and key paths without performing a TLS handshake (no network dependencies). The tests will run as part of `pytest tests` and cover the edge cases described in AC4."
  }
]
```
