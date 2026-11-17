# Issue Resolution: Support functools.partial callbacks in varnames / HookCaller.call_extra

## 1. Upstream Issue Reference
- **Repository:** `pytest-dev/pluggy`
- **Issue:** [#817](https://github.com/pytest-dev/pluggy/issues/817) - `call_extra() drops arguments for partial callbacks`
- **Status:** Open upstream

## 2. Problem Description
When passing a `functools.partial` callback to `HookCaller.call_extra()`, Pluggy dropped the remaining hook arguments.
- Callbacks requiring positional arguments failed with `TypeError: <function> missing required positional argument: <arg>`.
- Callbacks with default parameter values silently used the default instead of receiving the hook's arguments.

### Root Cause
In `src/pluggy/_decorators.py`, `varnames()` checked whether `func` is a routine with `inspect.isroutine(func)`. For instances of `functools.partial`, `isroutine()` is `False`, leading `varnames()` to inspect the `__call__` attribute via `func = getattr(func, "__call__", func)`. Because `partial.__call__` is a C method-wrapper, accessing its code object `func.__code__` raised `AttributeError`, which was caught and caused `varnames()` to return `(), ()`.
As a result, Pluggy concluded the callback accepted zero arguments and called it without forwarding any caller kwargs.

## 3. Reproduction Test Case
Added tests in `testing/test_helpers.py` (`test_varnames_partial`) and `testing/test_hookcaller.py` (`test_call_extra_partial`).

```python
def test_call_extra_partial(hc: HookCaller) -> None:
    from functools import partial

    def on_event(context: str, arg: object) -> tuple[str, object]:
        return context, arg

    callback = partial(on_event, "demo")
    assert hc.call_extra([callback], {"arg": 7}) == [("demo", 7)]
```

Prior to the fix:
`TypeError: test_call_extra_partial.<locals>.on_event() missing 1 required positional argument: 'arg'`

## 4. Solution & Code Changes
1. Added helper `_varnames_from_signature(func)` in `src/pluggy/_decorators.py` to extract positional and keyword parameter names using `inspect.signature()`.
2. Handled `functools.partial` in `varnames()`:
   - When `isinstance(func, functools.partial)` directly,
   - When an object's `__call__` is a `functools.partial`,
   - When `inspect.unwrap()` uncovers a wrapped `functools.partial`.
3. Verified all remaining parameters (both required positional and defaulted kwargs) are correctly detected and forwarded during hook execution.

## 5. Verification
- Reproduction tests pass.
- All 228 test cases across the entire test suite pass (`.venv/bin/pytest`).
- Ruff linter passes with zero warnings (`.venv/bin/ruff check src testing`).
