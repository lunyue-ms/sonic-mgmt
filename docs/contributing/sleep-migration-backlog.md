# time.sleep Migration Backlog

Personal tracking file (fork: fancuman/sonic-mgmt).
Do not submit upstream — entries move to PRs, not to the upstream repo.

See `time-sleep-migration.md` for the replacement guide and code templates.

## Overview

| # | 文件 | 行号 | sleep 时长 | 优先级 |
|---|------|------|-----------|--------|
| 1 | `tests/bgp/test_bgp_session.py` | 137 | 1s | Medium |
| 2 | `tests/fdb/test_fdb_mac_learning.py` | 272 | 30s | High |
| 3 | `tests/acl/test_stress_acl.py` | 90 | 60s | High |
| 4 | `tests/bgp/bgp_helpers.py` | 203 | 20s | High |
| 5 | `tests/bgp/test_bgp_stress_link_flap.py` | 234 | 5s | Medium |
| 6 | `tests/fdb/test_fdb_mac_move.py` | 135 | 变量 | High |
| 7 | `tests/vxlan/test_vxlan_ecmp.py` | 226 | 4s | Medium |
| 8 | `tests/vxlan/test_vxlan_ecmp.py` | 361 | 5s+1s | 待调查 |
| 9 | `tests/bgp/test_bgp_stress_link_flap.py` | 214 | 5s | Medium |
| 10 | `tests/vxlan/test_vxlan_route_advertisement.py` | 300 | 10s×N | 待调查 |

## Status Key

- `[ ]` Not started
- `[~]` In progress — branch: noted inline
- `[x]` PR submitted upstream — PR number noted
- `[X]` Merged upstream

---

## High Priority (clear state-change patterns)

- [ ] `tests/fdb/test_fdb_mac_learning.py:272` — `time.sleep(30)` after shutdown interfaces, then
  checks MAC entries are cleared. 30s wasted every run.
  Fix: `wait_until()` checking `not fdb_table_has_dummy_mac_for_interface(duthost, port)`.

- [ ] `tests/acl/test_stress_acl.py:90` — `time.sleep(60)` after writing ACL rules, then reads
  rule output. 60s wasted every run.
  Fix: `wait_until()` checking that target rules appear in `show acl rule {table_name}`.

- [ ] `tests/bgp/bgp_helpers.py:203` — `time.sleep(20)` after `wait_until(_dump_file_exists)`.
  Redundant sleep on top of an already-passing wait_until.
  Fix: Remove sleep; poll `parse_exabgp_dump()` returns non-empty routes.

- [ ] `tests/fdb/test_fdb_mac_move.py:135` — `time.sleep(FDB_POPULATE_SLEEP_TIMEOUT)` immediately
  before a `wait_until()` that checks FDB MAC count.
  Fix: Remove sleep; pass the constant as the `delay` parameter to `wait_until()`.

---

## Medium Priority

- [ ] `tests/bgp/test_bgp_stress_link_flap.py:234` — `time.sleep(5)` after `sonic-cfggen` writes
  BGP Monitor config. Should poll config DB for the key instead.

- [ ] `tests/bgp/test_bgp_stress_link_flap.py:214` — `time.sleep(5)` after writing BGP Sentinel
  config. Same pattern as above.

- [ ] `tests/vxlan/test_vxlan_ecmp.py:226` — `time.sleep(4)` after `setup_crm_interval()`, then
  reads CRM resources. Should poll `get_crm_resources()` for valid data.

- [ ] `tests/bgp/test_bgp_session.py:137` — `time.sleep(1)` after `no_shutdown`, immediately
  before `wait_until(check_bgp_session_state)`. Remove sleep; use `delay=1` parameter.

---

## Needs Investigation (may be legitimate sleeps)

- [ ] `tests/vxlan/test_vxlan_ecmp.py:361` — multiple `time.sleep()` after `redis-cli del`.
  Investigate: is there a queryable state after key deletion, or is this a required settle time?

- [ ] `tests/vxlan/test_vxlan_route_advertisement.py:300` — manual retry loop with `time.sleep(10)`.
  Replace entire loop with `wait_until()`. Verify command used for polling.

---

## Completed

| File | Upstream PR | Notes |
|------|-------------|-------|
| `tests/acl/test_acl_outer_vlan.py` | sonic-net/sonic-mgmt#22978 | Merged 2026-03-26, used as reference example |
