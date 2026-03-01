# BlackRoad Firmware

> **Edge device runtime, OTA firmware management, and Pi fleet orchestration for the BlackRoad OS platform.**

## Status: 🟢 GREEN LIGHT — Production Ready

**Last Updated:** 2026-03-01 | **Maintained By:** BlackRoad OS, Inc. | **CEO:** Alexa Amundson

---

## 📑 Table of Contents

1. [Overview](#-overview)
2. [Architecture](#-architecture)
3. [Repository Structure](#-repository-structure)
4. [Quick Start](#-quick-start)
5. [Pi Agent](#-pi-agent)
   - [Installation](#installation)
   - [Configuration](#configuration)
   - [Environment Variables](#environment-variables)
   - [Systemd Service](#systemd-service)
6. [Firmware Manager](#-firmware-manager)
   - [CLI Reference](#cli-reference)
7. [Components](#-components)
   - [Connection Manager](#connection-manager)
   - [Task Executor](#task-executor)
   - [Scheduler](#scheduler)
   - [Telemetry](#telemetry)
   - [Sensors & GPIO](#sensors--gpio)
   - [OTA Update Manager](#ota-update-manager)
8. [BlackRoad OS Ecosystem](#-blackroad-os-ecosystem)
9. [Testing](#-testing)
10. [Contributing](#-contributing)
11. [License](#-license)

---

## 🌌 Overview

**BlackRoad Firmware** is the edge-device layer of the BlackRoad OS platform — an AI-native operating system designed to enable entire organizations to run exclusively on AI agents. This repository provides:

| Component | Description |
|-----------|-------------|
| **Pi Agent** | Async Python runtime that connects Raspberry Pi (and other Linux edge devices) to the BlackRoad OS operator via WebSocket |
| **Firmware Manager** | CLI tool for tracking, verifying, and OTA-deploying firmware versions across the Pi fleet |
| **Sensors / GPIO** | Async GPIO controller with hardware and mock modes |
| **OTA Update Manager** | Manages over-the-air firmware update workflows |
| **Scheduler** | Min-heap task scheduler supporting one-shot and recurring jobs |
| **Telemetry** | Real-time CPU, memory, disk, temperature, and network metrics |

**Core Product:** API layer above Google, OpenAI, and Anthropic  
**Purpose:** Manage AI model memory and continuity  
**Goal:** Enable entire companies to operate exclusively by AI  

---

## 🏗️ Architecture

```
BlackRoad OS Operator  (cloud / on-prem)
        │  WebSocket (wss://)
        ▼
┌──────────────────────────────────┐
│           Pi Agent               │
│  ┌────────────┐  ┌────────────┐  │
│  │ Connection │  │  Executor  │  │
│  │  Manager   │  │ (shell /   │  │
│  └────────────┘  │  python /  │  │
│  ┌────────────┐  │  service)  │  │
│  │ Scheduler  │  └────────────┘  │
│  └────────────┘  ┌────────────┐  │
│  ┌────────────┐  │ Telemetry  │  │
│  │  Sensors / │  │ Collector  │  │
│  │    GPIO    │  └────────────┘  │
│  └────────────┘                  │
└──────────────────────────────────┘
        │
        ▼
  Raspberry Pi Hardware
  (OS · Kernel · Bootloader)
        │
        ▼
  Firmware Manager  ──►  SQLite DB
  (list / check / update / verify / log)
```

---

## 📁 Repository Structure

```
firmware/
├── README.md                  # This file
├── CONTRIBUTING.md            # Brand guidelines and contribution rules
├── TRAFFIC_LIGHT_SYSTEM.md    # Project status indicator reference
├── LICENSE                    # Proprietary license
│
├── src/
│   └── firmware_manager.py    # Firmware Manager CLI
│
├── pi_agent/                  # Pi Agent runtime package
│   ├── __init__.py
│   ├── main.py                # Entry point & PiAgent orchestrator
│   ├── config.py              # Configuration loader (JSON + env vars)
│   ├── connection.py          # WebSocket connection manager
│   ├── executor.py            # Task executor (shell, python, service …)
│   ├── scheduler.py           # Min-heap task scheduler
│   ├── telemetry.py           # System metrics collector
│   ├── ota/
│   │   ├── __init__.py
│   │   └── update_manager.py  # OTA update workflow
│   └── sensors/
│       ├── __init__.py
│       ├── cpu_temp.py        # CPU temperature reader
│       └── gpio_controller.py # Async GPIO controller (hw + mock)
│
├── pi/                        # Raspberry Pi deployment helpers
│   ├── install-pi-agent.sh    # One-line installer script
│   ├── blackroad-agent.service # systemd unit file
│   └── pi-ops.service         # Pi operations systemd unit
│
└── tests/
    └── test_sensors.py        # Sensor / GPIO unit tests
```

---

## 🚀 Quick Start

### Prerequisites

| Requirement | Minimum Version |
|-------------|----------------|
| Python | 3.8+ |
| pip | 22+ |
| websockets | 11+ |
| psutil | 5.9+ (optional, for full telemetry) |

### Install dependencies

```bash
pip install websockets psutil
```

### Run the Pi Agent (development)

```bash
# Using environment variables
export BLACKROAD_OPERATOR_URL="ws://your-operator:8080/ws/agent"
export BLACKROAD_AGENT_ID="pi-dev-01"

python -m pi_agent
```

### Run the Firmware Manager

```bash
python src/firmware_manager.py list
python src/firmware_manager.py check
python src/firmware_manager.py update --dry-run
```

---

## 🤖 Pi Agent

The Pi Agent is an async Python daemon that runs on each edge device. It:

- Establishes a persistent WebSocket connection to the BlackRoad OS operator
- Registers the device with its capabilities (`shell`, `telemetry`, `file_read`, `file_write`, `service`)
- Executes tasks dispatched by the operator (shell commands, Python scripts, systemd service management)
- Sends periodic heartbeats with live system telemetry
- Handles automatic reconnection with exponential backoff

### Installation

**One-line install (Raspberry Pi / Debian):**

```bash
sudo BLACKROAD_OPERATOR_URL="wss://operator.blackroad.io/ws/agent" \
  bash <(curl -sSL https://raw.githubusercontent.com/BlackRoad-Hardware/firmware/main/pi/install-pi-agent.sh)
```

The installer:
1. Detects the platform (Raspberry Pi, Jetson, generic Linux)
2. Installs system dependencies
3. Creates `/opt/blackroad/pi-agent` with a Python virtualenv
4. Generates `/etc/blackroad/pi-agent.config.json`
5. Installs and starts the `blackroad-agent` systemd service

### Configuration

Configuration is loaded from the first file found (in order):

1. Path passed with `-c / --config`
2. `$BLACKROAD_PI_CONFIG` environment variable
3. `/etc/blackroad/pi-agent.config.json`
4. `~/.config/blackroad/pi-agent.config.json`
5. `./pi-agent.config.json`
6. Defaults + environment variable overrides

**Example `pi-agent.config.json`:**

```json
{
  "operator": {
    "url": "wss://operator.blackroad.io/ws/agent",
    "reconnect_interval": 5.0,
    "reconnect_max_attempts": 0,
    "ping_interval": 30.0,
    "ping_timeout": 10.0
  },
  "agent": {
    "agent_id": "pi-a1b2c3d4",
    "agent_type": "pi-node",
    "capabilities": ["shell", "telemetry", "file_read", "file_write", "service"],
    "hostname": "blackroad-pi-01",
    "tags": {
      "platform": "raspberry-pi",
      "location": "rack-A",
      "role": "edge"
    }
  },
  "telemetry": {
    "heartbeat_interval": 15.0,
    "metrics_interval": 60.0,
    "report_system_metrics": true
  },
  "executor": {
    "max_concurrent_tasks": 4,
    "task_timeout": 300.0,
    "allowed_commands": [],
    "blocked_commands": ["rm -rf /", "mkfs", "dd if=", "shutdown", "reboot"]
  },
  "logging": {
    "level": "INFO",
    "file": "/var/log/blackroad/blackroad-agent.log",
    "format": "[%(asctime)s] %(levelname)s %(name)s: %(message)s"
  }
}
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `BLACKROAD_OPERATOR_URL` | WebSocket URL of the operator | `ws://localhost:8080/ws/agent` |
| `BLACKROAD_AGENT_ID` | Override auto-generated agent ID | _(derived from Pi serial / MAC)_ |
| `BLACKROAD_AGENT_TYPE` | Agent role type | `pi-node` |
| `BLACKROAD_HOSTNAME` | Override hostname | _(system hostname)_ |
| `BLACKROAD_HEARTBEAT_INTERVAL` | Seconds between heartbeats | `15.0` |
| `BLACKROAD_LOG_LEVEL` | Log verbosity (`DEBUG`/`INFO`/`WARNING`/`ERROR`) | `INFO` |
| `BLACKROAD_PI_CONFIG` | Path to config JSON file | _(see search order above)_ |
| `BLACKROAD_BRANCH` | Git branch used by the installer | `main` |

### Systemd Service

```bash
# Check status
sudo systemctl status blackroad-agent

# View live logs
sudo journalctl -u blackroad-agent -f

# Restart
sudo systemctl restart blackroad-agent

# Stop
sudo systemctl stop blackroad-agent
```

---

## 🛠️ Firmware Manager

A CLI tool (`src/firmware_manager.py`) that maintains a local SQLite database of firmware versions for the Pi fleet and simulates OTA update workflows.

**Fleet devices:** `aria64`, `alice`, `blackroad-pi`  
**Tracked components:** `os`, `kernel`, `bootloader`

### CLI Reference

#### `list` — Display firmware versions

```bash
python src/firmware_manager.py list [--device DEVICE] [--component {os,kernel,bootloader}] [--status {current,available,deprecated,pending}]
```

#### `check` — Check for available updates

```bash
python src/firmware_manager.py check [--device DEVICE]
```

#### `update` — Apply OTA firmware updates

```bash
python src/firmware_manager.py update [--device DEVICE] [--component {os,kernel,bootloader}] [--dry-run]
```

| Flag | Description |
|------|-------------|
| `--device` | Target a single device (default: all) |
| `--component` | Target a single component (default: all) |
| `--dry-run` | Simulate without writing changes |

#### `verify` — Verify firmware checksums

```bash
python src/firmware_manager.py verify [--device DEVICE] [--component {os,kernel,bootloader}]
```

#### `log` — Show update history

```bash
python src/firmware_manager.py log [--limit N]
```

**Version statuses:**

| Status | Meaning |
|--------|---------|
| `current` | Running the latest version |
| `available` | An update is available |
| `deprecated` | Old version, update recommended |
| `pending` | Update queued |

---

## 🧩 Components

### Connection Manager

`pi_agent/connection.py`

Manages the WebSocket connection to the BlackRoad OS operator with:

- State machine: `DISCONNECTED → CONNECTING → CONNECTED → RECONNECTING`
- Message handler registry (`connection.on(type, handler)`)
- Outbound message queue (fire-and-forget `await connection.send(type, payload)`)
- Automatic reconnection with exponential backoff (max 60 s) and jitter
- Agent registration on connect (id, hostname, roles, capabilities)

### Task Executor

`pi_agent/executor.py`

Executes operator-dispatched tasks with concurrency limiting via asyncio semaphore.

**Supported task types:**

| Type | Description |
|------|-------------|
| `shell` | Run a shell command |
| `script` | Execute a script file |
| `python` | Run inline Python code |
| `file_read` | Read a file and return its contents |
| `file_write` | Write content to a file |
| `service` | Manage a systemd service (start/stop/restart/status/enable/disable) |

All commands are checked against a configurable blocklist before execution.

### Scheduler

`pi_agent/scheduler.py`

Priority-queue task scheduler backed by `heapq`.

- One-shot tasks (with optional delay)
- Recurring tasks (`repeat_interval`)
- Cancel and reschedule by `task_id`
- Callback-based execution (integrates with `Executor`)

### Telemetry

`pi_agent/telemetry.py`

Collects system metrics and sends them with every heartbeat.

**Metrics reported:**

| Metric | Source |
|--------|--------|
| CPU % | `psutil.cpu_percent` |
| Memory % / used / total | `psutil.virtual_memory` |
| Disk % / used / total | `psutil.disk_usage("/")` |
| Load average (1 / 5 / 15 min) | `os.getloadavg` |
| Temperature (°C) | `/sys/class/thermal/thermal_zone0/temp` |
| Network bytes sent / received | `psutil.net_io_counters` |
| Uptime (seconds) | boot time delta |

Falls back to `/proc`-based metrics when `psutil` is not installed.

### Sensors & GPIO

`pi_agent/sensors/`

| Module | Description |
|--------|-------------|
| `gpio_controller.py` | Async GPIO controller; uses `RPi.GPIO` when available, falls back to mock mode automatically |
| `cpu_temp.py` | Reads CPU temperature from the thermal zone |

**Mock mode** is activated automatically when `RPi.GPIO` is unavailable (CI, development machines), enabling full test coverage without hardware.

```python
from pi_agent.sensors.gpio_controller import GPIOController

ctrl = GPIOController()       # auto-detects hardware / mock
ctrl.setup_pin(18, "out")
await ctrl.set_pin(18, True)
value = await ctrl.read_pin(18)
await ctrl.blink(18, times=3, interval=0.5)
```

### OTA Update Manager

`pi_agent/ota/update_manager.py`

Handles the full OTA lifecycle:

1. Fetch update manifest from operator
2. Download firmware image
3. Verify SHA-256 checksum
4. Flash and reboot (or defer to maintenance window)
5. Report result back to operator

---

## 🌐 BlackRoad OS Ecosystem

This repository is one of **578 repositories** across 15 specialized BlackRoad organizations, designed to support **30,000 AI agents + 30,000 human employees** under a single operator.

### npm Packages

The BlackRoad OS web and CLI layers are distributed as npm packages. The firmware layer integrates with them via the operator WebSocket API — no npm dependency is required on the edge device itself.

```bash
# BlackRoad OS operator SDK (install on your management workstation)
npm install @blackroad/operator-sdk

# BlackRoad CLI
npm install -g @blackroad/cli
```

### Stripe Billing Integration

Enterprise access to the BlackRoad OS platform (including the operator that manages this firmware layer) is billed through Stripe. Subscription tiers control:

| Feature | Starter | Professional | Enterprise |
|---------|---------|-------------|------------|
| Pi Agents | Up to 5 | Up to 100 | Unlimited |
| AI model memory | 30 days | 1 year | Unlimited |
| OTA update concurrency | 1 at a time | 10 concurrent | Unlimited |
| SLA | — | 99.9 % uptime | 99.99 % uptime |
| Support | Community | Email | Dedicated |

To manage your subscription: **[blackroad.io/billing](https://blackroad.io/billing)**  
For enterprise licensing: **blackroad.systems@gmail.com**

---

## 🧪 Testing

```bash
# Install test dependencies
pip install pytest pytest-asyncio

# Run all tests
pytest tests/

# Run with verbose output
pytest tests/ -v
```

**Test coverage:**

| Test file | What it covers |
|-----------|---------------|
| `tests/test_sensors.py` | GPIO controller mock mode: pin setup, read/write, blink |

GPIO tests run in **mock mode** automatically — no Raspberry Pi hardware required.

---

## 🤝 Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for full guidelines.

**Brand colours (all UI/design work must follow these):**

| Colour | Hex |
|--------|-----|
| Hot Pink (primary accent) | `#FF1D6C` |
| Amber | `#F5A623` |
| Electric Blue | `#2979FF` |
| Violet | `#9C27B0` |
| Background | `#000000` |
| Text | `#FFFFFF` |

**Gradient:** `linear-gradient(135deg, #FF1D6C 38.2%, #F5A623 61.8%)`  
**Spacing scale (golden ratio):** 8 → 13 → 21 → 34 → 55 → 89 → 144 px  
**Typography:** SF Pro Display, -apple-system, sans-serif · line-height 1.618

---

## 📜 License

**Copyright © 2026 BlackRoad OS, Inc. All Rights Reserved.**

**PROPRIETARY AND CONFIDENTIAL**

This software is the proprietary property of BlackRoad OS, Inc. and is **NOT for commercial resale**.

| Permitted | Prohibited |
|-----------|-----------|
| ✅ Testing and evaluation | ❌ Commercial use or resale |
| ✅ Educational purposes | ❌ Redistribution without written permission |
| ✅ Internal enterprise use (licensed customers) | ❌ Derivative works without written permission |

For commercial licensing: **blackroad.systems@gmail.com**

See [LICENSE](LICENSE) for complete terms.
