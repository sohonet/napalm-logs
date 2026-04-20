# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

napalm-logs is a Python library that parses network device syslog messages into structured, vendor-agnostic objects following OpenConfig/IETF YANG models. Supported vendors: Juniper (junos), Arista (eos), Cisco (ios, iosxr, nxos), Dell (os9, os10), Huawei (huawei), and Brocade (netiron). Output is published via pluggable transports (ZMQ, Kafka, HTTP, log).

## Commands

### Install for development
```bash
pip install -r requirements-dev.txt
pip install -e .
```

### Run full test suite
```bash
py.test -vv
```

### Run a single test case
```bash
py.test -vv -k "test_config[junos-BGP_PREFIX_THRESH_EXCEEDED-default]"
```
Test parameter format: `os_name-error_name-test_case` matching the directory structure under `tests/config/`.

### Linting
Linting runs automatically with pytest via `--pylama` (configured in `setup.cfg`). Uses pylama with mccabe, pep8, pyflakes. Max line length: 120.

## Architecture

### Multi-Process Pipeline

Messages flow through a pipeline of separate OS processes connected by ZeroMQ IPC:

1. **Listener** (`listener_proc.py` + `listener/`) — receives raw syslog (UDP/TCP/Kafka/ZMQ)
2. **Server** (`server.py`) — routes messages to the correct device process by matching against each device OS's `init.yml` prefix patterns
3. **Device** (`device.py`, one process per vendor OS) — parses message body with compiled regex, maps extracted values to YANG model paths
4. **Publisher Proxy** (`pub_proxy.py`) — IPC fan-out from device processes to publishers
5. **Publisher** (`publisher.py` + `transport/`, one per transport) — serializes and publishes structured output

`base.py:NapalmLogs` is the orchestrator that spawns and manages all these processes. `auth.py` provides an optional NaCl-based authenticator worker for clients. `ext/six.py` is the bundled Python 2/3 compatibility layer.

### Plugin System

Four subsystems use a factory/lookup pattern: **listeners** (`listener/`), **transports** (`transport/`), **serializers** (`serializer/`), **buffers** (`buffer/`). Each has an `__init__.py` with a `*_LOOKUP` dict mapping names to classes/functions, and a `get_*()` factory function.

### Device Configuration (YAML)

Parsing rules live in `napalm_logs/config/{vendor_os}/`:

- **`init.yml`** — defines `prefixes`: regex patterns to extract syslog header fields (host, tag, processId, etc.). The server uses these to identify which OS a message belongs to.
- **`{ERROR_NAME}.yml`** — defines one or more `messages` entries, each with:
  - `tag`: syslog tag to match
  - `values`: named regex capture groups (supports `|int` type casting suffix)
  - `line`: template string with `{value_name}` placeholders that becomes a regex
  - `model`: target YANG model name (e.g., `openconfig-bgp`)
  - `mapping.variables`: maps YANG paths (using `//` as separator) to captured value names. Curly-braced values in paths (e.g., `{peer}`) are substituted from captured data.
  - `mapping.static`: constant values injected into the YANG output

### Test Structure

Tests are data-driven and auto-discovered from `tests/config/{os}/{error_name}/{test_case}/`:
- `syslog.msg` — raw syslog input
- `yang.json` — expected structured output

The test harness (`tests/test_config.py`) starts an actual napalm-logs engine, sends each syslog message via UDP, receives the parsed output via ZMQ, and compares against the expected JSON (timestamps are excluded from comparison). Tests also assert that every configured error has at least one test case.

### Adding Support for a New Error Type

1. Create `napalm_logs/config/{os}/{ERROR_NAME}.yml` with the message definition
2. Create `tests/config/{os}/{ERROR_NAME}/{test_case_name}/` with `syslog.msg` and `yang.json`
3. For a new vendor OS, also create `napalm_logs/config/{os}/init.yml` with prefix patterns

To bootstrap the expected `yang.json`: leave it empty, run the test, and the assertion failure will print the actual parsed output which you can use as the expected value.
