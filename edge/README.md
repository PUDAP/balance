# first-edge

PUDA NATS edge service for the Opentrons OT-2 robot.

Bridges the OT-2 HTTP REST API to the PUDA NATS message bus — receives commands from the cloud, executes them on the robot, and publishes telemetry back.

---

## Architecture

```
NATS Server (cloud / LAN)
        │  puda.{machine_id}.cmd.queue       (JetStream — inbound commands)
        │  puda.{machine_id}.cmd.response.*  (JetStream — command replies)
        │  puda.{machine_id}.tlm.*           (core NATS — telemetry)
        ▼
  ┌──────────────────────┐   HTTP REST   ┌─────────────────┐
  │  edge/main.py        │ ─────────────▶│  OT-2 Robot     │
  │  EdgeRunner(robot)   │               │  :31950         │
  └──────────────────────┘               └─────────────────┘
```

`EdgeRunner` dispatches incoming NATS commands by matching `command.name` to `Opentrons` method names and calling them with `**command.params` — no adapter layer required.

---

## Setup

```powershell
cd edge
copy .env.example .env
notepad .env
```

```dotenv
MACHINE_ID=opentrons
OPENTRONS_IP=192.168.0.10
NATS_SERVERS=nats://100.109.131.12:4222,nats://100.109.131.12:4223,nats://100.109.131.12:4224
```

| Variable | Description |
|---|---|
| `MACHINE_ID` | Unique identifier for this robot on the NATS bus |
| `OPENTRONS_IP` | IPv4 address of the OT-2 on the local network |
| `NATS_SERVERS` | Comma-separated list of NATS server URLs |

---

## Run

```powershell
# From repo root
uv sync
uv run --package opentrons-edge python edge/main.py
```

Or use the included batch file on Windows:

```powershell
.\edge\start_edge.bat
```

The service retries automatically on fatal errors (5 s backoff) and ignores `KeyboardInterrupt` to keep running in unattended environments.

---

## Available NATS commands

Send commands via `puda_comms.CommandService` with `machine_id` matching your `.env`.
Command names map directly to `Opentrons` method names.

| Command | Key params | Description |
|---|---|---|
| `upload_and_run` | `code` (str), `filename?`, `wait?`, `max_wait?`, `poll_interval?` | Upload and run a protocol |
| `get_status` | `run_id?` (str) | Get run status (latest run if omitted) |
| `pause` | `run_id` (str) | Pause a running protocol |
| `resume` | `run_id` (str) | Resume a paused protocol |
| `stop` | `run_id` (str) | Stop / cancel a run |
| `upload_labware` | `labware` (dict) | Upload a custom labware definition |
| `is_connected` | — | Check robot reachability |
| `get_labware_types` | — | List known labware load-names |
| `get_pipette_types` | — | List known pipette instrument names |

### Example

```python
import asyncio
from puda_comms import CommandService
from puda_comms.models import CommandRequest

async def run():
    async with CommandService(servers=["nats://100.109.131.12:4222"]) as svc:
        reply = await svc.send_queue_commands(
            requests=[
                CommandRequest(
                    name="upload_and_run",
                    machine_id="opentrons",
                    params={
                        "code": open("my_protocol.py").read(),
                        "wait": True,
                        "max_wait": 300,
                    },
                    step_number=1,
                )
            ],
            run_id="run-001",
            user_id="user1",
            username="Alice",
            timeout=360,
        )
        print(reply.response.status)   # SUCCESS / ERROR

asyncio.run(run())
```

---

## Telemetry

Every telemetry tick the edge publishes three subjects (replace `opentrons` with your `MACHINE_ID`):

| Subject | Payload |
|---|---|
| `puda.opentrons.tlm.heartbeat` | Heartbeat timestamp |
| `puda.opentrons.tlm.health` | `{"connected": true, "robot_ip": "192.168.0.10"}` |
| `puda.opentrons.tlm.state` | `{"robot_ip": "192.168.0.10"}` |

---

## Requirements

| Dependency | Version | Purpose |
|---|---|---|
| Python | >= 3.14 | — |
| `opentrons-driver` | >= 0.1.0 | Local OT-2 driver (workspace package) |
| `puda-comms` | == 0.0.10 | NATS `EdgeRunner` / `EdgeNatsClient` |
| `pydantic` | >= 2.12.5 | Settings model |
| `pydantic-settings` | >= 2.12.0 | `.env` loading |
| `python-dotenv` | >= 1.2.1 | `.env` file support |
