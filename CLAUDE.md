# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

sonic-mgmt is the test infrastructure and management automation repository for SONiC (Software for Open Networking in the Cloud). It contains thousands of pytest-based test cases for validating SONiC network switches, plus Ansible automation for testbed deployment and management.

## Commands

### Setup
```bash
python3 -m venv venv && source venv/bin/activate && pip install -r requirements.txt
# Or use the Docker container (recommended):
./setup-container.sh -n sonic-mgmt -d /data
```

### Running Tests
```bash
# Run a specific test against a testbed
cd tests
pytest test_feature.py -v --testbed=vms-kvm-t0 --inventory=../ansible/veos_vtb

# Run with topology marker filter
pytest -m "topology_t0" tests/bgp/test_bgp_fact.py

# Via run_tests.sh (more options)
./tests/run_tests.sh -n <testbed> -d <dut_name> -f <testbed_file> \
                     -i ansible/<inventory> -c <test_path>

# Via Make (Docker-based, wraps testbed-cli.sh)
make test T=bgp/test_bgp_fact.py TOPO=vms-kvm-t0
make add-topo TOPO=vms-kvm-t1
make remove-topo TOPO=vms-kvm-t1
```

### Linting
```bash
# Install pre-commit hooks (run once)
pip install pre-commit && pre-commit install-hooks

# Run checks on changed files (same as CI gate)
pre-commit run --from-ref <base_commit> --to-ref HEAD

# Run on staged files only
pre-commit
```

CI blocks merges on: flake8, pylint, mypy, black, isort failures, pytest collection errors, invalid test markers.

## Architecture

```
sonic-mgmt/
├── tests/               # Main pytest test suite
│   ├── common/          # Shared utilities, fixtures, device abstractions (legacy)
│   ├── common2/         # Redesigned utilities with stricter QA (type hints, black, mypy)
│   ├── conftest.py      # Root pytest fixtures (~174KB)
│   └── <feature>/       # One directory per SONiC feature (bgp/, acl/, vlan/, qos/, ...)
├── ansible/             # Testbed deployment and management
│   ├── testbed.yaml     # Topology definitions
│   ├── testbed-cli.sh   # CLI for add/remove topology, deploy minigraph
│   └── roles/           # Ansible roles for device configuration
├── spytest/             # Alternative SPyTest framework
├── sdn_tests/           # SDN-specific tests
├── test_reporting/      # Test result processing
└── docs/                # Setup and contribution guides
```

### Key Concepts

- **Topologies**: Tests are marked for specific topologies (`t0`, `t1`, `t2`, `dualtor`, `smartswitch`). A test marked `t1` will not run on a `t0` testbed.
- **DUT (Device Under Test)**: The SONiC switch; accessed via `duthost` fixture.
- **PTF (Packet Test Framework)**: Container used for data-plane packet injection/verification; accessed via `ptfhost` fixture.
- **Fixtures**: Defined in `conftest.py` at various directory levels. Key ones: `duthosts`, `duthost`, `tbinfo`, `ptfhost`, `rand_one_dut_hostname`, `enum_frontend_dut_hostname`, `fanouthosts`, `dpuhosts`.
- **SmartSwitch/DPU**: `duthost` is the NPU (main switch); `dpuhosts` is the list of DPU SONiC instances reachable via midplane. "Dark mode" = DPUs shut down; "Lit mode" = DPUs up.
- **Multi-ASIC**: Use `enum_asic_index` fixture to iterate over ASICs on multi-ASIC platforms.

### Code Quality Tiers

| Area | Tools enforced |
|------|---------------|
| `tests/` (general) | flake8 (max-line-length=120) |
| `tests/common2/` | flake8 + black + isort + mypy (strict) + pylint |
| `spytest/` | Relaxed flake8 |

New utilities should prefer `tests/common2/` with full type annotations, unit tests, and docstrings.

## Writing Tests

```python
import pytest
from tests.common.helpers.assertions import pytest_assert

@pytest.mark.topology('t0')
def test_my_feature(duthosts, rand_one_dut_hostname, tbinfo):
    """Test that my feature works correctly."""
    duthost = duthosts[rand_one_dut_hostname]

    duthost.shell('config my_feature enable')
    output = duthost.show_and_parse('show my_feature status')
    pytest_assert(output[0]['status'] == 'enabled', "Feature should be enabled")
```

### Critical Rules

- **Topology marker required**: Every test must have `@pytest.mark.topology(...)`.
- **No `time.sleep()`**: Use `wait_until()` helpers from `tests/common/utilities.py`.
- **Restore DUT state**: Tests must clean up after themselves (idempotent).
- **No hardcoded IPs/ports**: Use fixtures and `tbinfo` instead.
- **Fixture scope awareness**: Session-scoped fixtures persist across tests; function-scoped reset each test.
- **Flaky network tests**: Use `@pytest.mark.flaky(reruns=3)` where appropriate.

### PTF Data-Plane Testing

```python
from tests.common.plugins.ptfadapter import ptf_runner
ptf_runner(duthost, ptfhost, 'my_ptf_test',
           platform_dir='ptftests',
           params={'router_mac': router_mac})
```

## PR Requirements

- **Commit format**: `[component/test]: Description`
- **Signed-off-by**: Required (`git commit -s`)
- **CLA**: Sign Linux Foundation EasyCLA
- **PR body**: Use `.github/PULL_REQUEST_TEMPLATE.md` — fill in every section
- **Validation**: New tests must be validated on at least VS topology
