# Review feedback (architect_review, plan v2)

- The plan skips the architect, but the fix location is a design decision. Add the architect role and write an HLD before Dev.

Design constraint: change only super_len in src/requests/utils.py. When the stream is a text file object (io.TextIOBase, including StringIO), Content-Length must be the UTF-8 byte length, not the character count. Do not change prepare_body in src/requests/models.py.

Binary BytesIO must keep its current byte length. An explicit Content-Length from the caller must not be overwritten. Multipart and JSON bodies stay unchanged.

Update requirements.md and stories/01-demo-test.md so the change site is super_len only, and add an acceptance criterion: StringIO("héllo") prepares with Content-Length 6, while BytesIO("héllo".encode()) stays Content-Length 6.
