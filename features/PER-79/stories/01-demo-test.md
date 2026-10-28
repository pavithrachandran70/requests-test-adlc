# Story — Update from_key_val_list examples to the Python 3.12 OrderedDict repr

---

## Metadata

- **ID**: PER-79
- **Type**: story
- **Status**: planned
- **Epic**: _(none)_
- **Covers**: AC-1, AC-2, AC-3
- **Estimate**: 1 point
- **Track**: quick

---

## Description

The docstring examples in `from_key_val_list` (`src/requests/utils.py`) currently show the old Python 2-era `OrderedDict` repr:

```python
OrderedDict([('key', 'val')])
```

On Python 3.12, the same calls now produce:

```python
OrderedDict({'key': 'val'})
```

Two docstring lines are affected: the list example and the dict example. The `ValueError` example in that docstring is already correct and must not be changed.

Because pytest is configured with `--doctest-modules`, these stale lines cause doctest failures when running:

```bash
python -m pytest --doctest-modules src/requests/utils.py
```

**What we're doing**: Update both docstring lines to match the Python 3.12 repr. No behavior or logic changes to `from_key_val_list` itself.

**Focus**: The two docstring lines showing `OrderedDict` repr; no behavior or logic changes.

---

## Technical Specification

### File to Change

- **Path**: `src/requests/utils.py`
- **Function**: `from_key_val_list`

### Current Docstring (excerpt)

```python
def from_key_val_list(value):
    """Take an object and test to see if it can be represented as a
    dictionary. Unless it can not be represented as such, return an
    OrderedDict, e.g.,

    ::

        >>> from_key_val_list([('key', 'val')])
        OrderedDict([('key', 'val')])
        >>> from_key_val_list({'key': 'val'})
        OrderedDict([('key', 'val')])
        >>> from_key_val_list('string')
        Traceback (most recent call last):
        ...
        ValueError: cannot encode objects that are not 2-tuples
    """
```

### Required Changes

Replace the two lines showing `OrderedDict([('key', 'val')])` with `OrderedDict({'key': 'val'})`:

**Line 1** (list example):
```python
# Before
        OrderedDict([('key', 'val')])
# After
        OrderedDict({'key': 'val'})
```

**Line 2** (dict example):
```python
# Before
        OrderedDict([('key', 'val')])
# After
        OrderedDict({'key': 'val'})
```

**Do NOT change**:
- The `ValueError` example (already correct)
- Any logic or behavior in `from_key_val_list`
- Any other functions or docstrings in `src/requests/utils.py`

### Expected Docstring (excerpt, after change)

```python
        >>> from_key_val_list([('key', 'val')])
        OrderedDict({'key': 'val'})
        >>> from_key_val_list({'key': 'val'})
        OrderedDict({'key': 'val'})
        >>> from_key_val_list('string')
        Traceback (most recent call last):
        ...
        ValueError: cannot encode objects that are not 2-tuples
```

---

## Acceptance Criteria

### AC-1: Docstring examples match Python 3.12 repr

**Given** the `from_key_val_list` function in `src/requests/utils.py`  
**When** the docstring examples are evaluated on Python 3.12  
**Then** both the list example and the dict example display `OrderedDict({'key': 'val'})` instead of the old `OrderedDict([('key', 'val')])` form

**Verification**:
- Open `src/requests/utils.py`
- Locate the `from_key_val_list` docstring
- Confirm both output lines show `OrderedDict({'key': 'val'})`

---

### AC-2: Doctest passes for from_key_val_list

**Given** pytest is configured with `--doctest-modules`  
**When** running `python -m pytest --doctest-modules src/requests/utils.py`  
**Then** the `from_key_val_list` docstring examples pass without failure

**Verification**:
```bash
python -m pytest --doctest-modules src/requests/utils.py
```
- Exit code 0
- No doctest failures reported for `from_key_val_list`

---

### AC-3: Function behavior unchanged

**Given** the `from_key_val_list` function receives input as a list of tuples or a dict  
**When** the function is called with `[('key', 'val')]` or `{'key': 'val'}`  
**Then** it returns an `OrderedDict` with the same contents as before, maintaining backward compatibility

**Verification**:
- Run existing unit tests for `from_key_val_list` in `tests/test_utils.py` (if present)
- Manually verify in a Python 3.12 REPL:
  ```python
  from requests.utils import from_key_val_list
  result = from_key_val_list([('key', 'val')])
  assert isinstance(result, OrderedDict)
  assert result == OrderedDict([('key', 'val')])
  ```

---

## Out of Scope

- Changing the logic or behavior of `from_key_val_list`
- Modifying the `ValueError` example line (already correct)
- Updating any other docstrings or functions in `src/requests/utils.py`
- Changing test files or pytest configuration
- Framework, ORM, or cloud stack decisions

---

## Notes

- This is a **docstring-only** fix targeting two specific lines
- The change aligns docstring output with Python 3.12's `OrderedDict.__repr__`
- No functional code changes are required
- The `ValueError` example is already in the correct format and must remain unchanged

---

## References

- **Ticket**: PER-79
- **Plan**: `per-79-plan.md`
- **Requirements**: `requirements.md` (AC-1, AC-2, AC-3)
- **File**: `src/requests/utils.py`
- **Function**: `from_key_val_list`
