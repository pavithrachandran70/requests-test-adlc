# Review feedback (architect_review, plan v4)

- 1. On the test snippets (Test 1–3)

HTTPAdapter.cert_verify is cert_verify(self, conn, url, verify, cert). These samples call adapter.cert_verify(None, missing_cert_path, None), so the missing path is passed as url and cert is never set. Rewrite Test 1, Test 2, and Test 3 to call cert_verify with a stand-in connection, an https URL, verify=True, and cert set to the missing path or the (cert_path, missing_key_path) tuple. Assert exc.filename is that path. The tests must not open a TLS connection.
