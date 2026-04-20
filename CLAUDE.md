# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

napalm-logs parses syslog messages from network devices (Juniper, Arista, Cisco, Dell, Huawei, Brocade) and converts them to structured, vendor-agnostic data following OpenConfig/IETF YANG models. Output is transported via ZMQ, Kafka, HTTP, or logging.

## Common Commands

```bash
# Install in development mode
pip install -e .
pip install -r requirements-dev.txt

# Run all tests
pytest -vv

# Run a single parametrized test (pattern: os-ERROR_TYPE-case)
pytest tests/test_config.py::test_config[junos-BGP_PREFIX_THRESH_EXCEEDED-default] -vv

# Lint
pylama napalm_logs/
```

## Architecture

Multi-process pipeline connected via ZMQ IPC:

```
Listener → Server → Device Processes → Publisher → External Clients
(UDP/TCP/   (routes to   (per-OS regex    (ZMQ/Kafka/
 Kafka)      device by    parsing, YANG     HTTP output)
             prefix)      generation)
```

**Key source files:**
- `napalm_logs/base.py` — Main `NapalmLogs` engine: initializes and manages all sub-processes
- `napalm_logs/server.py` — Routes incoming syslog messages to the correct device process by matching OS-specific prefixes
- `napalm_logs/device.py` — Per-OS worker: loads YAML config, applies regex patterns, extracts fields, maps to YANG models
- `napalm_logs/listener_proc.py` + `napalm_logs/listener/` — Syslog input (UDP, TCP, Kafka)
- `napalm_logs/publisher.py` + `napalm_logs/transport/` — Output transports (ZMQ, Kafka, HTTP, Log)
- `napalm_logs/auth.py` — Optional NaCl-based client authentication

**Config-driven message parsing** (`napalm_logs/config/{os_name}/`):
- Each OS has a directory with YAML files defining error types
- YAML files contain: regex patterns for matching/extracting fields, and YANG model mappings
- Adding support for a new syslog message = adding a new YAML config file

## Test Structure

Tests are fixture-based and auto-discovered from filesystem structure:

```
tests/config/{os_name}/{ERROR_TYPE}/{test_case}/
    syslog.msg    — raw syslog input
    yang.json     — expected YANG-structured output
```

The test harness (`tests/test_config.py`) spins up a NapalmLogs engine, sends syslog messages via UDP, receives parsed output via ZMQ, and compares against expected YANG JSON. Tests are parametrized by `[os-ERROR_TYPE-case]`.

## Code Style

- Max line length: 120 characters
- Linters: pylama (mccabe, pep8, pyflakes)
- Python 2/3 compatible (uses `napalm_logs/ext/` compatibility layer)
