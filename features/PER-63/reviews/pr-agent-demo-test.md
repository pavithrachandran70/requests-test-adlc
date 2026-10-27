```json
[
  {
    "lens": "adversarial",
    "location": "src/requests/adapters.py:367-370",
    "trigger_condition": "FileNotFoundError constructor receives three positional args (errno, message, filename) but Python's FileNotFoundError does not accept errno as first positional argument in its standard signature",
    "guard_snippet": "Use FileNotFoundError(message) and set .filename attribute explicitly, or use two-arg form FileNotFoundError(errno, message) which Python will map to .errno and .strerror, then set .filename separately",
    "potential_consequence": "Runtime TypeError or malformed exception at call site; errno.ENOENT ends up in .errno but message goes to .strerror and filename to an unintended attribute, breaking AC1's contract that .filename holds the missing path"
  },
  {
    "lens": "adversarial",
    "location": "src/requests/adapters.py:373-376",
    "trigger_condition": "Same three-arg FileNotFoundError constructor issue as above (missing key path raise)",
    "guard_snippet": "Use FileNotFoundError(message) and set .filename attribute explicitly, or use two-arg form FileNotFoundError(errno, message) which Python will map to .errno and .strerror, then set .filename separately",
    "potential_consequence": "Runtime TypeError or malformed exception at call site; AC2's contract (.filename holds missing key path) breaks"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:21-34",
    "trigger_condition": "Test asserts exc.errno == errno.ENOENT and exc.filename == missing_cert_path but does not verify the message text matches the expected format from requirements",
    "guard_snippet": "Add assertion: assert 'Could not find the TLS certificate file, invalid path' in str(exc) or check exc.strerror contains expected message components",
    "potential_consequence": "AC1 requires both .filename attribute and human-readable message; test passes even if message is garbled or empty"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:37-60",
    "trigger_condition": "Test asserts exc.errno == errno.ENOENT and exc.filename == missing_key_path but does not verify the message text matches the expected format from requirements",
    "guard_snippet": "Add assertion: assert 'Could not find the TLS certificate key file, invalid path' in str(exc) or check exc.strerror contains expected message components",
    "potential_consequence": "AC2 requires both .filename attribute and human-readable message; test passes even if message is garbled or empty"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:63-77",
    "trigger_condition": "Test catches 'except OSError' in a bare try/except and sets caught_as_oserror=True, but does not verify the exception type is FileNotFoundError before catching",
    "guard_snippet": "Restructure to 'except FileNotFoundError as e: caught_as_fnfe=True; assert isinstance(e, OSError); caught_as_oserror=True' or verify e.__class__.__name__ == 'FileNotFoundError' inside the OSError handler",
    "potential_consequence": "Test passes if *any* OSError subclass is raised (e.g. PermissionError, IsADirectoryError) rather than specifically FileNotFoundError, violating AC1/AC2"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:37-60",
    "trigger_condition": "Temporary cert file created with delete=False but cleanup only happens in finally block; if pytest exits early (e.g. unhandled exception before finally) the file leaks",
    "guard_snippet": "Use context manager: with tempfile.NamedTemporaryFile(..., delete=False) as f: ... valid_cert_path = f.name; adapter.cert_verify(...); os.unlink(valid_cert_path) or use delete=True and read f.name before exiting context",
    "potential_consequence": "/tmp pollution if test suite is interrupted; minor resource leak"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:21-34",
    "trigger_condition": "Test does not verify that the exception message format matches the old OSError message format for backward compatibility (callers may parse the message string)",
    "guard_snippet": "Assert message contains both 'Could not find the TLS certificate file' and 'invalid path' and the cert path itself, matching the old f-string format",
    "potential_consequence": "Callers that parse the old 'Could not find the TLS certificate file, invalid path: <path>' message will break if the new exception's str() differs"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_adapters.py:37-60",
    "trigger_condition": "Test does not verify that the exception message format matches the old OSError message format for backward compatibility (callers may parse the message string)",
    "guard_snippet": "Assert message contains both 'Could not find the TLS certificate key file' and 'invalid path' and the key path itself, matching the old f-string format",
    "potential_consequence": "Callers that parse the old 'Could not find the TLS key file, invalid path: <path>' message will break if the new exception's str() differs"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_requests.py:979-980",
    "trigger_condition": "Changed pytest.raises(IOError) to pytest.raises(FileNotFoundError) but AC3 says existing tests should remain compatible or be updated only where they assert exact type; this is an exact-type change but no comment explains why IOError→FileNotFoundError was necessary",
    "guard_snippet": "Leave as pytest.raises(IOError) (FileNotFoundError is subclass, will still catch) or add comment 'Updated to FileNotFoundError per AC1/AC2 contract change'",
    "potential_consequence": "Violates AC3's guidance to minimize test changes; the old pytest.raises(IOError) would still pass, making this change unnecessary and potentially masking other issues"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_requests.py:981-983",
    "trigger_condition": "Changed assertion from exact string match to 'in str(e.value)' but does not verify .filename attribute per AC1 contract",
    "guard_snippet": "Add assertion: assert e.value.filename == INVALID_PATH",
    "potential_consequence": "AC1 explicitly requires .filename attribute; integration test does not verify it, allowing a malformed exception to pass"
  },
  {
    "lens": "adversarial",
    "location": "tests/test_requests.py:985-991",
    "trigger_condition": "Changed assertion from exact string match to 'in str(e.value)' but does not verify .filename attribute per AC2 contract",
    "guard_snippet": "Add assertion: assert e.value.filename == INVALID_PATH",
    "potential_consequence": "AC2 explicitly requires .filename attribute; integration test does not verify it, allowing a malformed exception to pass"
  },
  {
    "lens": "edge-case-hunter",
    "location": "src/requests/adapters.py:367-370",
    "trigger_condition": "cert_file is an empty string (os.path.exists('') returns False)",
    "guard_snippet": "Guard: if conn.cert_file and conn.cert_file.strip() and not os.path.exists(conn.cert_file): raise ... (or add explicit empty-string check before existence check)",
    "potential_consequence": "FileNotFoundError raised with filename='' which is misleading; caller cannot distinguish 'user passed empty string' from 'path genuinely missing'"
  },
  {
    "lens": "edge-case-hunter",
    "location": "src/requests/adapters.py:373-376",
    "trigger_condition": "key_file is an empty string (os.path.exists('') returns False)",
    "guard_snippet": "Guard: if conn.key_file and conn.key_file.strip() and not os.path.exists(conn.key_file): raise ... (or add explicit empty-string check before existence check)",
    "potential_consequence": "FileNotFoundError raised with filename='' which is misleading; caller cannot distinguish 'user passed empty string' from 'path genuinely missing'"
  },
  {
    "lens": "edge-case-hunter",
    "location": "src/requests/adapters.py:367-370",
    "trigger_condition": "cert_file is a directory that exists (os.path.exists returns True for directories)",
    "guard_snippet": "Guard: if conn.cert_file and not os.path.isfile(conn.cert_file): raise FileNotFoundError(...) or ValueError('cert path is not a file')",
    "potential_consequence": "Existing directory passed as cert path bypasses the exists() check, leading to TLS handshake failure with cryptic SSL error downstream rather than clear FileNotFoundError"
  },
  {
    "lens": "edge-case-hunter",
    "location": "src/requests/adapters.py:373-376",
    "trigger_condition": "key_file is a directory that exists (os.path.exists returns True for directories)",
    "guard_snippet": "Guard: if conn.key_file and not os.path.isfile(conn.key_file): raise FileNotFoundError(...) or ValueError('key path is not a file')",
    "potential_consequence": "Existing directory passed as key path bypasses the exists() check, leading to TLS handshake failure with cryptic SSL error downstream rather than clear FileNotFoundError"
  },
  {
    "lens": "edge-case-hunter",
    "location": "src/requests/adapters.py:367-370",
    "trigger_condition": "cert_file path contains null bytes or other invalid characters that os.path.exists accepts but filesystem rejects",
    "guard_snippet": "Wrap os.path.exists in try/except ValueError (null byte in path) or OSError (invalid path format) and reraise as FileNotFoundError with original path in .filename",
    "potential_consequence": "os.path.exists raises ValueError or OSError with confusing message instead of clear FileNotFoundError; caller cannot catch the expected exception type"
  },
  {
    "lens": "edge-case-hunter",
    "location": "src/requests/adapters.py:373-376",
    "trigger_condition": "key_file path contains null bytes or other invalid characters that os.path.exists accepts but filesystem rejects",
    "guard_snippet": "Wrap os.path.exists in try/except ValueError (null byte in path) or OSError (invalid path format) and reraise as FileNotFoundError with original path in .filename",
    "potential_consequence": "os.path.exists raises ValueError or OSError with confusing message instead of clear FileNotFoundError; caller cannot catch the expected exception type"
  },
  {
    "lens": "edge-case-hunter",
    "location": "src/requests/adapters.py:367-370",
    "trigger_condition": "cert_file is a symlink pointing to a non-existent target (os.path.exists follows symlinks and returns False)",
    "guard_snippet": "Current behavior is correct (broken symlink is effectively missing file); document this as expected in AC1 or add explicit test case",
    "potential_consequence": "No bug, but undocumented edge case; callers may expect different error for 'symlink exists but target missing' vs 'path does not exist'"
  },
  {
    "lens": "edge-case-hunter",
    "location": "tests/test_adapters.py:21-34",
    "trigger_condition": "Test does not verify behavior when cert path is an empty string",
    "guard_snippet": "Add test case: adapter.cert_verify(conn, url, True, '') and assert either FileNotFoundError with filename='' or ValueError('empty cert path')",
    "potential_consequence": "Edge case not covered by tests; implementation may raise FileNotFoundError with filename='' (misleading) or other exception"
  },
  {
    "lens": "edge-case-hunter",
    "location": "tests/test_adapters.py:37-60",
    "trigger_condition": "Test does not verify behavior when key path is an empty string",
    "guard_snippet": "Add test case: adapter.cert_verify(conn, url, True, (valid_cert, '')) and assert either FileNotFoundError with filename='' or ValueError('empty key path')",
    "potential_consequence": "Edge case not covered by tests; implementation may raise FileNotFoundError with filename='' (misleading) or other exception"
  },
  {
    "lens": "edge-case-hunter",
    "location": "tests/test_adapters.py:21-34",
    "trigger_condition": "Test does not verify behavior when cert path is a directory that exists",
    "guard_snippet": "Add test case: create temp directory, pass its path as cert, assert FileNotFoundError or appropriate exception (not silent pass-through to TLS handshake)",
    "potential_consequence": "Edge case not covered; implementation may accept directory path and fail later with cryptic SSL error"
  },
  {
    "lens": "verification-gap",
    "location": "tests/test_adapters.py (missing test)",
    "trigger_condition": "AC5 requires 'No behavior change when files exist' but no test verifies valid cert/key paths still work after the change",
    "guard_snippet": "Add test: create valid temp cert and key files, call adapter.cert_verify(conn, url, True, (cert_path, key_path)), assert no exception raised and conn.cert_file/conn.key_file are set correctly",
    "potential_consequence": "AC5 verification missing; regression in valid-path handling would not be caught by test suite"
  },
  {
    "lens": "verification-gap",
    "location": "tests/test_adapters.py (missing test)",
    "trigger_condition": "AC6 requires 'CA-bundle lookup unchanged' but no test verifies the CA-bundle OSError raise remains untouched",
    "guard_snippet": "Add test: mock or arrange invalid CA-bundle path, call adapter.cert_verify(..., verify=invalid_ca_path), assert OSError (not FileNotFoundError) is raised",
    "potential_consequence": "AC6 verification missing; accidental change to CA-bundle exception type would not be caught"
  },
  {
    "lens": "verification-gap",
    "location": "tests/test_requests.py:977-991",
    "trigger_condition": "Integration tests updated to expect FileNotFoundError but do not verify .errno attribute per AC1/AC2",
    "guard_snippet": "Add assertions: assert e.value.errno == errno.ENOENT in both test cases",
    "potential_consequence": "AC1/AC2 contract partially untested; .errno could be None or wrong value and integration tests would still pass"
  }
]
```
