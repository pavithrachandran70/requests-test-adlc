Fix these forge PR review comments (auto-route):

### demo-test comment 6469488496
Required code change. The head commit is empty, so this pull request changes no files and the bug is still present.

In src/requests/models.py, Request._encode_files must coerce every multipart field name to str before it is passed to urllib3. An int key in data or files must become the field name "1".

Keep the current urlencoded body for data={1: "a"} with no files (body 1=a). Do not change file-tuple shapes or method signatures.

Add a unit test that calls prepare() with no network connection and covers an int key in data when files is set, and an int key in files. python -m pytest tests must pass for that test and the existing request-preparation tests.

This is behavior inside story 01-demo-test. The acceptance criteria already describe it. Implement that story in code.
