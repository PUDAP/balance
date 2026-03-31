# opentrons

Monorepo for the Opentrons OT-2 driver and PUDA edge service.

Communicates directly with the OT-2 REST API over HTTP — no Opentrons App, no MCP server, no Docker required.

---

## Repository layout

```
opentrons/
├── driver/          # opentrons-driver — high-level OT-2 Python driver
│   ├── opentrons.py         # Opentrons class (unified API)
│   ├── protocol.py          # Protocol/ProtocolCommand models & code generation
│   ├── labware/             # Custom labware JSON definitions
│   │   ├── mass_balance_vial_30000.json
│   │   └── mass_balance_vial_50000.json
│   ├── experiments/
│   │   └── viscosity_optimization/
│   ├── pyproject.toml
│   └── README.md            # Driver API reference
│
├── edge/            # first-edge — PUDA NATS edge service
│   ├── main.py              # Config + EdgeRunner wiring + telemetry loop
│   ├── .env.example         # Environment variable template
│   ├── pyproject.toml
│   └── README.md            # Edge service docs
│
├── pyproject.toml   # uv workspace root (members: driver, edge)
└── uv.lock
```

---

## Packages

| Package | Name | Description |
|---|---|---|
| `driver/` | `opentrons-driver` | High-level OT-2 HTTP wrapper — upload protocols, control runs, manage labware |
| `edge/` | `opentrons-edge` | PUDA edge service — bridges the OT-2 to the NATS message bus |

---

## Quick start

### Driver only

```python
from driver.opentrons import Opentrons

robot = Opentrons(robot_ip="192.168.0.10")
robot.startup()

result = robot.upload_and_run(open("my_protocol.py").read())
print(result["run_status"])   # "succeeded" / "failed" / "stopped"
```

### Edge service

```powershell
# 1. Configure environment
cd edge
copy .env.example .env
notepad .env
```

```dotenv
MACHINE_ID=opentrons
OPENTRONS_IP=192.168.0.10
NATS_SERVERS=nats://100.109.131.12:4222,nats://100.109.131.12:4223,nats://100.109.131.12:4224
```

```powershell
# 2. Install and run (from repo root)
uv sync
uv run --package opentrons-edge python edge/main.py
```

---

## Requirements

| Dependency | Version |
|---|---|
| Python | >= 3.14 |
| uv | latest |

---

## See also

- [`driver/README.md`](driver/README.md) — driver API reference, protocol builder, run control, custom labware
- [`edge/README.md`](edge/README.md) — NATS edge service setup, commands, telemetry
