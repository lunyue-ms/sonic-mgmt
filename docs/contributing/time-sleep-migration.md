# Replacing time.sleep() with wait_until() in sonic-mgmt tests

## Why This Matters

Using `time.sleep()` to wait for state changes causes two problems:

1. **Flaky tests**: A fixed sleep may be too short on a slow device and too long on a fast one.
2. **Slow tests**: The test always waits the full duration even when the condition is met in 1 second.

The project rule (see `CLAUDE.md`) is explicit: **No `time.sleep()`** — use `wait_until()` from
`tests/common/utilities.py` instead.

---

## wait_until() Signature

```python
from tests.common.utilities import wait_until

wait_until(timeout, interval, delay, condition, *args, **kwargs)
```

| Parameter   | Meaning |
|-------------|---------|
| `timeout`   | Maximum seconds to wait before giving up |
| `interval`  | Seconds between each poll attempt |
| `delay`     | Seconds to wait before the first poll (replaces a preceding `time.sleep`) |
| `condition` | A callable that returns `True` (done) or `False` (keep waiting) |
| `*args/kwargs` | Passed through to `condition` on every call |

**Behavior**: returns `True` if condition is met within timeout; returns `False` on timeout.
Exceptions raised inside `condition` are caught, logged, and treated as `False`.

---

## Decision Matrix: Replace or Keep?

| Situation | Action |
|-----------|--------|
| Waiting for a state change (BGP up, ACL counter ready, MAC entry cleared) | **Replace** with `wait_until()` |
| Waiting for a show command output to match a value | **Replace** with `wait_until()` |
| Manual retry loop: `while retry > 0: ... time.sleep(N) ...` | **Replace** the whole loop with `wait_until()` |
| `time.sleep()` immediately before a `wait_until()` call | **Remove** the sleep; use the `delay` parameter instead |
| Brief mandatory protocol delay with no queryable state | Keep (document why) |
| Inside `wait_until()` internals (`utilities.py`) | Keep — this is the implementation |
| `spytest/` directory | Different quality tier; check with that team |

---

## Code Templates

### Template 1: Wait for state change

```python
def _check_<state>(duthost, ...):
    result = duthost.show_and_parse('show ...')
    # Return True when condition is met, False to keep polling
    # Do NOT call pytest.fail() here — just return False
    return <condition on result>

pytest_assert(
    wait_until(60, 2, 0, _check_<state>, duthost, ...),
    "Expected state was not reached within 60s"
)
```

### Template 2: Wait for counter/metric to appear, then read its value

```python
def _counter_ready():
    result = duthost.show_and_parse('show ...')
    return any(r['key'] == target for r in result)

if not wait_until(timeout, 2, 0, _counter_ready):
    pytest.fail("Counter not ready for {}".format(target))

# Read value only after confirming it exists
result = duthost.show_and_parse('show ...')
value = next(r['value'] for r in result if r['key'] == target)
```

---

## Reference Example: test_acl_outer_vlan.py (PR #22978)

This is the canonical before/after for this project.

**Before** (`tests/acl/test_acl_outer_vlan.py`):
```python
# Wait for orchagent to update the ACL counters
time.sleep(timeout)
result = duthost.show_and_parse('aclshow -a')

if len(result) == 0:
    pytest.fail("Failed to retrieve acl counter for {}|{}".format(table_name, rule_name))
```

**After**:
```python
def _check_acl_counter():
    result = duthost.show_and_parse('aclshow -a')
    if len(result) == 0:
        return False
    for rule in result:
        if table_name == rule['table name'] and rule_name == rule['rule name']:
            return True
    return False

if not wait_until(timeout, 2, 0, _check_acl_counter):
    pytest.fail("Failed to retrieve acl counter for {}|{}".format(table_name, rule_name))

result = duthost.show_and_parse('aclshow -a')
```

---

## Common Pitfalls

- **Don't call `pytest.fail()` inside the condition function** — return `False` instead and let the
  caller decide how to handle timeout.
- **Timeout sizing**: start with `existing_sleep_value * 3` as a safe timeout; use `interval=2`.
- **The `delay` parameter** replaces a `time.sleep()` that precedes a `wait_until()` loop, not the
  polling interval itself.
- **Don't wrap `wait_until()` in another loop** — it already retries internally.

---

## Current Inventory

As of 2026-03-27: ~741 occurrences across ~250 files in `tests/`.

To check remaining work:

```bash
grep -rn "time\.sleep" tests/ | grep -v "common/utilities.py" | grep -v ".pyc"
```

Upstream is actively reducing this count — check recent commits before starting work on a file.
