---
key: PER-79
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Update from_key_val_list examples to the Python 3.12 OrderedDict repr
---


The docstring examples in `from_key_val_list` (`src/requests/utils.py`) still show the old form:

```
OrderedDict([('key', 'val')])
```

On Python 3.12 the same calls print `OrderedDict({'key': 'val'})`. Both stale lines are still in the file, at the list example and the dict example. The `ValueError` example in that docstring is already correct. Pytest is configured with `--doctest-modules`, so these two lines fail a doctest run.

Expected behavior. Both examples show `OrderedDict({'key': 'val'})`. The function still returns an `OrderedDict` with the same contents.

Scope. Edit those two docstring lines only. Do not change `from_key_val_list` behavior or the `ValueError` example.

Acceptance criteria.

* The docstring matches `repr(from_key_val_list([('key', 'val')]))` and `repr(from_key_val_list({'key': 'val'}))` on Python 3.12.
* `python -m pytest --doctest-modules src/requests/utils.py` passes for `from_key_val_list`.
* The return value is still an `OrderedDict` with the same contents.
