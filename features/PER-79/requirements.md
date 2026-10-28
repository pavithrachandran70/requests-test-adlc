# Requirements — PER-79

**Update from_key_val_list examples to the Python 3.12 OrderedDict repr**

---

## Acceptance Criteria

### AC-1: Docstring examples match Python 3.12 repr

**Given** the `from_key_val_list` function in `src/requests/utils.py`  
**When** the docstring examples are evaluated on Python 3.12  
**Then** both the list example and the dict example display `OrderedDict({'key': 'val'})` instead of the old `OrderedDict([('key', 'val')])` form

---

### AC-2: Doctest passes for from_key_val_list

**Given** pytest is configured with `--doctest-modules`  
**When** running `python -m pytest --doctest-modules src/requests/utils.py`  
**Then** the `from_key_val_list` docstring examples pass without failure

---

### AC-3: Function behavior unchanged

**Given** the `from_key_val_list` function receives input as a list of tuples or a dict  
**When** the function is called with `[('key', 'val')]` or `{'key': 'val'}`  
**Then** it returns an `OrderedDict` with the same contents as before, maintaining backward compatibility

---

## Out of Scope

- Changing the logic or behavior of `from_key_val_list`
- Modifying the `ValueError` example line (already correct)
- Updating any other docstrings or functions in `src/requests/utils.py`
- Changing test files or pytest configuration
- Framework, ORM, or cloud stack decisions

---

## Notes

- This is a documentation-only fix targeting two specific docstring lines
- The change aligns docstring output with Python 3.12's `OrderedDict` repr
- No functional code changes are required
