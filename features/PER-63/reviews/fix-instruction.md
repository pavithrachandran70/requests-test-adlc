Fix these forge PR review comments (auto-route):

### demo-test comment 6441627532
Please address the review findings by making the required changes in these two files:

1. `src/requests/adapters.py` — Update `HTTPAdapter.cert_verify` to raise `FileNotFoundError(errno.ENOENT, ..., path)` for both missing certificate and key paths, ensuring the exception's `filename` attribute is set correctly.

2. `tests/test_adapters.py` — Add unit tests covering both missing certificate and missing key paths, verifying the exception type and `filename` attribute.

Please remove the unrelated `.gitignore` changes so the PR stays within the scope of PER-63. Ensure the relevant tests pass before updating the PR.

### demo-test comment 4388544199
Please address the review findings by making the required changes in these two files:

src/requests/adapters.py — Update HTTPAdapter.cert_verify to raise FileNotFoundError(errno.ENOENT, ..., path) for both missing certificate and key paths, ensuring the exception's filename attribute is set correctly.

tests/test_adapters.py — Add unit tests covering both missing certificate and missing key paths, verifying the exception type and filename attribute.

Please remove the unrelated .gitignore changes so the PR stays within the scope of PER-63. Ensure the relevant tests pass before updating the PR.
