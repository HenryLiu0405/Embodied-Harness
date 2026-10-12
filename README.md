# Embodied-Harness

**An agent harness for visually grounded robot tasks on the Unitree G1.**

Embodied-Harness connects a cloud vision-language planner to robot perception,
navigation, and manipulation through a local execution runtime built on ROS 2
Humble. The agent grounds natural-language instructions in scene observations
and generates semantic task plans. The harness validates those plans, dispatches
robot skills, maintains mission state, and returns action evidence for
closed-loop verification and replanning.

The current integration connects FoundationPose, Navigation, and the G1 motion
Gateway. A web console provides natural-language task submission, monitoring,
intervention, and replay. Task execution belongs to the local Supervisor and
does not depend on an operator advancing each stage in the browser.

## Demo

https://github.com/user-attachments/assets/59934c96-b3b6-48cb-a8c2-f17d752b9a03

## System Architecture

The **agent** makes grounded planning and verification decisions. The
**harness** supplies the observation context, semantic tools, execution
constraints, persistent state, and feedback loop around those decisions.

```text
Natural-language instruction
        |
        v
Web Console / Task API
        |
        v
Planner Agent + Closed-loop Controller <--> Model Gateway <--> Cloud VLM
        ^       |
        |       v
        |  GoalSpec + PlanGraph / PlanRevision
        |       |
        |       v
        |  Execution Supervisor <---- Local stop / drop interrupt
        |       |
        |       v
        |  Semantic Skill Registry + Shared Dispatcher
        |       |
        |       v
        |  ROS 2 Adapters --> Navigation / FoundationPose / Gateway --> G1
        |       |
        +-------+-- Observations, World State, and action results
```

| Layer | Responsibility | Implementation |
|---|---|---|
| Planner and model gateway | Ground instructions against observed objects, generate structured semantic plans, and validate model responses against the current observation and world version. | [`planner_agent.py`](phi_robot/autonomy/planner_agent.py), [`model_gateway.py`](phi_robot/autonomy/model_gateway.py) |
| Execution runtime | Own the active mission, execute plans in the background, persist action history, and manage interruption and reconciliation. | [`runtime.py`](phi_robot/autonomy/runtime.py), [`supervisor.py`](phi_robot/autonomy/supervisor.py), [`ledger.py`](phi_robot/autonomy/ledger.py) |
| Semantic skills and adapters | Expose task-level tools such as `move_to_location`, `select_object`, `pick_object`, and `verify_placement`; translate them into existing robot interfaces. | [`agent_tools.py`](phi_robot/autonomy/agent_tools.py), [`skills/`](phi_robot/skills/), [`ros_bridge.py`](phi_robot/autonomy/ros_bridge.py) |
| Observations and world state | Combine timestamped visual evidence, object identities, symbolic destinations, and robot state for grounded decisions. | [`observations.py`](phi_robot/autonomy/observations.py), [`objects.py`](phi_robot/autonomy/objects.py), [`locations.py`](phi_robot/autonomy/locations.py), [`world_state.py`](phi_robot/autonomy/world_state.py) |
| Verification and recovery | Evaluate action outcomes, request fresh evidence, revise the unexecuted plan tail, and coordinate transport-drop recovery after confirmed local stop. | [`closed_loop.py`](phi_robot/autonomy/closed_loop.py), [`recovery.py`](phi_robot/autonomy/recovery.py) |
| Console and replay | Submit instructions and inspect missions, plan revisions, model decisions, action results, events, and metrics. | [`AutonomyConsole.jsx`](phi_robot_fronted/src/AutonomyConsole.jsx), [`service.py`](phi_robot/autonomy/service.py), [`evaluation.py`](phi_robot/autonomy/evaluation.py) |

### Execution Boundaries

- **Grounded goals:** `ObjectRef` separates logical object identity from
  perception IDs; `LocationRef` keeps destinations symbolic. The current
  executable destination resolver supports the nine-cell grid.
- **Controlled physical execution:** agent actions use validated semantic plans
  and the Supervisor. Autonomous and legacy developer actions share one
  dispatcher and per-robot writer lock in the deployed composition. The model
  does not receive raw ROS, joint, velocity, or service-administration authority.
- **Explicit uncertainty:** stale observations are rejected, unknown physical
  outcomes require reconciliation, and replanning preserves dispatched action
  history rather than silently retrying it.
- **Local interruption:** stop requests use an independent interrupt lane. A
  transport-drop event requests local stop before model-based diagnosis;
  recovery requires confirmed stop and a resolved prior action.
- **Traceable decisions:** durable ledgers link model calls, frame hashes, plan
  revisions, skill invocations, and events for inspection and replay.

The architecture has offline/replay test coverage; end-to-end cloud-driven
missions and transport-drop recovery still require real-robot acceptance
testing. See the [planner](docs/autonomy/phase5/implementation.md),
[recovery](docs/autonomy/phase7/implementation.md), and
[console/replay](docs/autonomy/phase8/implementation.md) validation notes.

## Repository Structure

```text
Embodied-Harness/
├── phi_robot/                          # Backend Python package
│   ├── autonomy/                      # Agent harness and local execution runtime
│   │   ├── planner_agent.py           # Natural-language grounding and semantic planning
│   │   ├── model_gateway.py           # Structured model calls and offline replay provider
│   │   ├── gemini_provider.py         # Gemini multimodal provider adapter
│   │   ├── runtime.py                 # Per-robot runtime ownership
│   │   ├── supervisor.py              # Background plan execution and interruption
│   │   ├── agent_tools.py             # Planner-visible semantic skill contracts
│   │   ├── ros_bridge.py              # Observation and semantic ROS adapters
│   │   ├── closed_loop.py             # Verification and plan-tail revision
│   │   ├── recovery.py                # Transport-drop recovery coordination
│   │   ├── ledger.py                  # Durable mission, action, and event history
│   │   ├── evaluation.py              # Replay bundles and mission metrics
│   │   └── deployment.py              # Shared agent / legacy deployment composition
│   ├── skills/                        # Versioned registry, dispatcher, and compatibility layer
│   ├── adapters/
│   │   └── ros_acceptance.py           # ROS2 adapter for Gateway/Nav/FP integration
│   ├── api_server.py                   # Flask API server (REST + SSE)
│   ├── step_debug.py                   # Step-by-step controller (StepDebugController)
│   ├── dev_console.py                  # Debug console controller
│   ├── mission_planner.py              # Legacy grid-to-step task planner
│   ├── mission_runner.py               # Task execution engine
│   ├── mission_service.py              # Task persistence
│   ├── robot_state.py                  # SafetyStateMachine (9 states + allowed transitions)
│   ├── robot_profile.py                # Multi-robot profile management
│   ├── service_manager.py              # Remote systemd service management
│   ├── service_registry.json           # Service registry (workstation + Orin)
│   ├── robots.json                     # Robot IP/domain configuration
│   ├── scene_coords.json               # Scene coordinates in the Nav2 map frame
│   ├── audit.py                        # Operation audit logs
│   ├── recovery_diag.py                # Recovery diagnostic event factory
│   └── models.py                       # Data models
├── phi_robot_fronted/                  # React 18 + Vite 5 frontend
│   ├── index.html                      # Main page entry point
│   ├── debug.html                      # Debug console entry point
│   └── src/
│       ├── App.jsx                     # Autonomy console by default; legacy grid via ?developer=1
│       ├── AutonomyConsole.jsx         # Instructions, readiness, mission monitoring, and replay
│       ├── debug-main.jsx              # Debug page entry point
│       ├── DeveloperConsole.jsx        # Debug console
│       ├── StepDebugPanel.jsx          # Step-by-step debug panel
│       ├── LockBoard2D.jsx             # Nine-cell grid canvas
│       ├── UnifiedConsole.jsx          # Unified console
│       ├── StepDetailPanel.jsx         # Step details + terminal output
│       └── index.css                   # Lunar theme styles
├── bridge/                             # Integration bridges
│   ├── workstation/
│   │   └── goal_bridge.py              # /start_navigation → /goal_pose (waits for /navigation_complete)
│   └── orin/
│       ├── goal_bridge.py              # Orin-side goal bridge
│       ├── action_bridge.py            # Action bridge
│       └── zmq_bridge.py               # ZMQ bridge between DDS domains
├── systemd/                            # systemd service unit files
│   ├── workstation/                    # Deploy to ~/.config/systemd/user/ on the workstation
│   └── orin/                           # Deploy to ~/.config/systemd/user/ on the robot
├── scripts/                            # Startup scripts
│   ├── start_taskmanger.sh
│   └── start_foundationpose.sh
├── robot_interfaces/                   # ExecuteTrajectory service definition
├── phi_robot_interfaces/               # Custom phi_robot interfaces
├── run_autonomy_api.py                 # Agent harness entry point (explicit opt-in)
├── run_phi_robot_api.py                # Legacy backend entry point
├── run_ros_acceptance.sh               # ROS2 integration startup script
└── requirements.txt                    # Python dependencies
```

## Requirements

| Component | Version | Notes |
|------|------|------|
| OS | Ubuntu 22.04 | Workstation + Orin |
| ROS2 | Humble | `/opt/ros/humble` |
| Python | 3.10 | System Python or conda |
| Node.js | ≥ 18 | Frontend |
| DDS | Fast-DDS | Workstation |
| DDS | CycloneDDS | Orin robot |
| conda | miniconda3 | FoundationPose environment |

### Python Dependencies

```text
flask>=3.0,<4.0
flask-cors>=4.0,<5.0
jsonschema>=4.23,<5.0
```

`rclpy`, `std_msgs`, and `sensor_msgs` are provided by ROS2 Humble.

Use the repository's `environment.yml` to create an isolated development environment:

```bash
conda env create -f environment.yml
conda activate AgenticRobot
python --version  # Must be 3.10.x
```

`AgenticRobot` includes the backend and offline test dependencies. ROS Python
packages are supplied by ROS2 Humble on Ubuntu 22.04. For the formal offline
acceptance gate, source Humble on the target Ubuntu host, then run
`TASKMANAGER_PYTHON="$CONDA_PREFIX/bin/python" ./scripts/check_offline.sh`.

### Frontend Dependencies

React 18 + Vite 5 + Ant Design 5

## Configuration

### Gemini Robotics-ER 2 (Autonomous Runtime, Explicit Opt-in)

Edit `.env` in the project root. It is ignored by Git and loaded automatically by
the runtime:

```dotenv
GEMINI_API_KEY=your_rotated_api_key
GEMINI_ROBOTICS_MODEL=gemini-robotics-er-1.6-preview
GEMINI_API_MODE=generate_content
PHI_AUTONOMY_ENABLED=1
TASKMANAGER_PYTHON=/home/hairo/miniconda3/envs/AgenticRobot/bin/python
PHI_OPERATOR_CIDRS=127.0.0.0/8,192.168.50.0/24
```

The model and API mode have defaults. Refer to Google's official model
documentation for Robotics-ER availability, and override the model ID if your
account provides a different one. Set `PHI_AUTONOMY_ENABLED=1` on the target host
to select the autonomous entry point through the existing systemd startup
scripts. The general-purpose `gemini-flash-latest` model is not Robotics-ER.
Constructing `GeminiAutonomyRuntime` alone does not make network or robot calls.
When opening the console from another computer, add its trusted management
subnet to `PHI_OPERATOR_CIDRS`; otherwise, monitoring remains available, but
pause and emergency-stop requests will be rejected.

### 1. Robot Configuration (`phi_robot/robots.json`)

The **single source of truth** for robot profiles. `switch_robot.sh` reads this
file to generate `active_robot.env`.

```json
{
  "active": "A",
  "robots": {
    "A": {
      "label": "Robot A",
      "ip": "192.168.3.168",
      "domain_id": 66,
      "camera_host": "192.168.3.168",
      "orin_host": "unitree@192.168.3.168"
    },
    "B": {
      "label": "Robot B",
      "ip": "192.168.3.215",
      "domain_id": 5,
      "camera_host": "192.168.3.215",
      "orin_host": "unitree@192.168.3.215"
    }
  }
}
```

**Fields**:

| Field | Description |
|------|------|
| `ip` | Robot IP address |
| `domain_id` | ROS2 DDS domain; use a separate domain for each robot to avoid interference |
| `camera_host` | ZMQ camera server IP address |
| `orin_host` | SSH connection string for remote management by `service_manager` |

### 2. Scene Coordinates (`phi_robot/scene_coords.json`)

Configure nine destination cells and one stock point in the Nav2 map frame.

```json
{
  "frame": "nav2_map",
  "stock_point": { "x": -2.5, "y": 1.9, "theta": 2.2689 },
  "grid_cells": {
    "nw": { "x": -0.4, "y": 1.4, "theta": -1.5708 },
    "n":  { "x": 0.11, "y": 2.0, "theta": -1.5708 },
    "ne": { "x": 0.9,  "y": 2.8, "theta": -1.5708 },
    "w":  { "x": -1.1, "y": 2.0, "theta": -1.5708 },
    "c":  { "x": -0.5, "y": 2.7, "theta": -1.5708 },
    "e":  { "x": 0.2,  "y": 3.5, "theta": -1.5708 },
    "sw": { "x": -1.8, "y": 2.7, "theta": 0.0 },
    "s":  { "x": -1.1, "y": 3.5, "theta": 1.5708 },
    "se": { "x": -0.5, "y": 4.1, "theta": 1.5708 }
  },
  "map_bounds": { "x_min": -3.0, "x_max": 1.5, "y_min": 1.0, "y_max": 4.6 },
  "approach_offset": 0.6
}
```

To update the coordinates, edit this file and restart TaskManger. The frontend
loads them automatically from `/api/scene_coords`.

### 3. `active_robot.env` (Generated Automatically)

Generated by `switch_robot.sh` from `robots.json`. **Do not edit it manually**:

```dotenv
ROS_DOMAIN_ID=66
APP_DOMAIN=66
WAIC_ROBOT_ID=A
WAIC_ROBOT_IP=192.168.3.168
WAIC_CAMERA_HOST=192.168.3.168
FP_CAMERA_HOST=192.168.3.168
WAIC_ORIN_HOST=unitree@192.168.3.168
```

All systemd services load this file through `EnvironmentFile` to select the DDS
domain and target robot.

### Deploying to a WAIC Host (Configuration Only)

Maintain these three configuration files on the host; no Python or frontend code
changes are required:

1. Root `.env`: API key, `PHI_AUTONOMY_ENABLED=1`, and the TaskManger Python path.
2. `phi_robot/robots.json`: robot IP addresses, ROS domains, camera hosts, and Orin hosts.
3. `phi_robot/scene_coords.json`: nine-cell grid coordinates for the current map.

After syncing the code and building the frontend, run a configuration check that
does not contact ROS, cloud services, or the robot:

```bash
cd /home/hairo/waic/TaskManger
${TASKMANAGER_PYTHON} run_autonomy_api.py --check-config
```

Once the check returns `"ok": true`, the existing `waic-taskmanger.service`
continues to invoke `scripts/start_taskmanger.sh`. The script reads the same
`.env` and `active_robot.env`. In autonomous mode, Flask serves the built frontend
on the same port, so a long-running Vite development server is unnecessary. No
additional configuration is needed when host paths and ROS service names match
the WAIC setup. Override different paths or service names with `PHI_*` variables
in `.env`.

Before enabling real-robot operation, complete static interface checks, individual
action trials, full-mission trials, and injected-drop hardware-in-the-loop (HIL)
tests in that order. Stop confirmation uses `/nav_pause` together with consecutive,
fresh `/odom` samples showing low velocity. Insufficient evidence leaves the
system awaiting operator intervention; successful publication alone does not
prove that the robot has stopped.

## Quick Start (Legacy ROS Setup)

The commands below start the legacy ROS backend and developer tools. For the
agent harness entry point, see the [autonomous runtime configuration](#configuration)
and its deployment validation requirements.

### 1. Clone the Repository

```bash
git clone https://github.com/HenryLiu0405/Embodied-Harness.git
cd Embodied-Harness
```

### 2. Build the ROS2 Interface Packages

```bash
source /opt/ros/humble/setup.bash
colcon build --symlink-install --packages-select robot_interfaces phi_robot_interfaces
source install/setup.bash
```

### 3. Install Dependencies

```bash
# Python virtual environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Frontend
cd phi_robot_fronted
npm install
npm run build        # Production build in dist/
```

### 4. Configure the Robot

```bash
# Set the robot IP address and domain_id in robots.json
vim phi_robot/robots.json

# Generate active_robot.env
python3 -c "
import sys; sys.path.insert(0, '.')
from phi_robot import robot_profile
robot_profile.write_env_file(robot_profile.get_active())
"
```

### 5. Start the System

**Option A: systemd (recommended for deployment)**

```bash
cp systemd/workstation/*.service ~/.config/systemd/user/
# Update paths in the .service files for your environment
systemctl --user daemon-reload
systemctl --user start waic-taskmanger
```

**Option B: manual startup**

```bash
# Terminal 1: backend
source /opt/ros/humble/setup.bash
source install/setup.bash
source /home/hairo/waic/active_robot.env
source .venv/bin/activate
python3 run_phi_robot_api.py --port 5000 --acceptance ros \
  --path-plan-service /start_navigation

# Terminal 2: frontend
cd phi_robot_fronted
npm run dev
```

### 6. Open the Web Interface

| Page | URL | Purpose |
|------|------|------|
| Autonomy console | `http://localhost:4173/` | Natural-language tasks, readiness, mission monitoring, and replay; requires the autonomous backend |
| Developer page | `http://localhost:4173/?developer=1` | Legacy grid route selection and live step progress |
| Debug console | `http://localhost:4173/debug.html` | Legacy step-by-step debugging / automatic execution (FP select_target) |

## Legacy Developer Workflow

The grid and step-debug interfaces remain available for compatibility and
diagnostics. They are separate from the default natural-language console and
do not define the Agent's planning loop.

```text
Developer page (?developer=1)          Debug console (debug.html)
        │                                      │
Draw a grid route and click RUN        Inspect the loaded plan
        │                                      │
POST /api/dev/step_debug/load          Click automatic execution
        │                                      │
Plan loaded for display               StepDebugController._run_auto()
        │                             (select_target + fail-stop)
        │                                      │
        ◀──────── Live progress via SSE /stream ┘
        │
UnifiedConsole: progress markers
StepDetailPanel: step status
```

**Legacy behavior**: the developer-page RUN workflow and debug-console automatic execution
share the same execution engine, `StepDebugController._run_auto()`, including
automatic single-object locking through FP `select_target`. The developer page
receives live step updates through SSE without polling.

## Switching Robots

```bash
# Switch to robot B
/home/hairo/waic/switch_robot.sh B

# Check the active robot
curl http://localhost:5000/api/robot/active
```

`switch_robot.sh` performs the following steps:

1. Read the target robot profile from `robots.json`.
2. Regenerate `active_robot.env`.
3. Restart `waic-navigation`, `waic-goal-bridge`, and `waic-foundationpose`.
4. Restart `waic-taskmanger` last to apply the new domain.

> **Required**: `ROS_DOMAIN_ID` is fixed when a process starts. Switching robots
> changes the domain and requires restarting the full service stack.

## ROS2 Interfaces

### Services Called by TaskManger

| Service | Type | Description |
|------|------|------|
| `/start_navigation` | `ExecuteTrajectory` | Path-planning navigation (→ goal_bridge → /goal_pose) |
| `/submit_carry_task` | `SubmitCarryTask` | Submit a lifting task (single-object pose → Gateway) |
| `/set_lay_down` | `SetBool` | Place the box down |
| `/set_stand` | `Trigger` | Move the robot to a standing posture |
| `/notify_goal_reached` | `SetBool` | Notify Gateway of arrival (transition from carry_walking to goal_reached_locked) |
| `/foundationpose/select_target` | `SelectTarget` | FP single-object locking (MODE_TARGET/MODE_DROPPED) |
| `/foundationpose/activate` | `Activate` | Enable/disable FP tracking |
| `/foundationpose/reset` | `Reset` | Reset the FP tracker |

### Topics Published or Subscribed to by TaskManger

| Topic | Type | Direction | Description |
|------|------|------|------|
| `/locomotion_mode` | `Int32` | Publish | Motion mode: `1` = normal walking, `2` = carrying; published every 0.4 s |
| `/nav_pause` | `Bool` | Publish | Pause/resume navigation |
| `/nav_reached` | `Bool` | Subscribe | Navigation arrival confirmation (Nav2 → ZMQ Bridge → TaskManger) |
| `/foundationpose/pose_results` | `PoseEstimateArray` | Subscribe | FP pose estimates in the pelvis frame |
| `/fp_state` | `String` | Subscribe | FP tracking state as JSON |
| `/odom` | `Odometry` | Subscribe | Odometry; uses odom rather than the deprecated TF path |
| `/G1Env/env_state_act` | `ByteMultiArray` | Subscribe | SONIC environment state, including waist joint angles; used only by FP |

### SelectTarget Service Details

```text
/foundationpose/select_target
  Request:
    mode: 0 (MODE_TARGET: pick) | 1 (MODE_DROPPED: drop detection)
    select: true=enter mode, false=exit mode
    material_points_xy: [x1, y1, x2, y2, ...]  # All stock points
    target_points_xy: [x1, y1, x2, y2, ...]    # All destination points
    pick_x, pick_y: Coordinates of the stock point to pick from
    point_tolerance: Position tolerance (default: 0.3 m)

  Response:
    success: bool
    matched_object_id: int32    # Matched object ID; -1 means no match
    box_world_xy: [x1, y1, ...] # World coordinates of all visible boxes
    box_object_ids: [id1, ...]  # Corresponding box IDs
```

**Automatic execution**: before the pick step,
`select_target(select=True, MODE_TARGET)` locks onto the box nearest the stock
point. After picking, `select_target(select=False)` exits target mode and
re-enables drop detection.

## DDS & ZMQ Bridge Topology

```text
Robot Orin                      Workstation
CycloneDDS domain 0              Fast-DDS domain 66/5

  /goal_pose    ──ZMQ:5561──▶     /goal_pose
  /cmd_vel      ──ZMQ:5562──▶     /cmd_vel
  /tf           ──ZMQ:5564──▶     /tf
  /tf_static    ──ZMQ:5565──▶     /tf_static
  /nav_pause    ◀──ZMQ:5566──     /nav_pause
  /nav_reached  ──ZMQ:5567──▶     /nav_reached
```

Twelve workers (six subscriber/publisher pairs) run on the Orin through
`/home/unitree/waic_bridge/zmq_bridge.py`.

## API Endpoints

### Tasks & Status

| Method | Path | Description |
|------|------|------|
| GET | `/api/health` | Health check |
| GET | `/api/snapshot` | System snapshot |
| GET | `/api/robot/state` | Current SafetyStateMachine state |

### Step-by-Step Debugging (StepDebugController)

| Method | Path | Description |
|------|------|------|
| POST | `/api/dev/step_debug/load` | Load a plan (main-page RUN button) |
| POST | `/api/dev/step_debug/auto_run` | Start automatic execution (debug-console execution button) |
| POST | `/api/dev/step_debug/pause` | Pause execution |
| POST | `/api/dev/step_debug/resume` | Resume execution |
| POST | `/api/dev/step_debug/stop` | Stop execution |
| POST | `/api/dev/step_debug/select_target_toggle` | Manually toggle FP single-target mode |
| POST | `/api/dev/step_debug/drop_check` | Run a manual drop check |
| GET | `/api/dev/step_debug/stream` | SSE event stream for live main-page progress |
| GET | `/api/dev/step_debug/state` | Current step-debug state |

### Robot Management

| Method | Path | Description |
|------|------|------|
| GET | `/api/robot/active` | Active robot and domain synchronization status |
| GET | `/api/robot/list` | All available robots |
| GET | `/api/scene_coords` | Scene coordinates for dynamic frontend rendering |

### Service Management

| Method | Path | Description |
|------|------|------|
| GET | `/api/services` | List all service statuses |
| POST | `/api/services/<id>/start` | Start a service |
| POST | `/api/services/<id>/stop` | Stop a service |

## SafetyStateMachine

Nine states, an allowed-transition list, and per-action blocked states prevent
invalid operations.

| State | Description |
|------|------|
| `IDLE` | Idle, not ready |
| `STANDING` | Standing and ready to move |
| `MOVING` | Walking |
| `ARRIVED` | Arrived at the destination |
| `PICKING` | Lifting in progress |
| `HOLDING` | Holding a box |
| `PLACING` | Placement in progress |
| `EMERGENCY` | Emergency stop |
| `ERROR` | Fault |

**Blocked actions**:

| Action | Blocked States |
|------|------|
| walk | PICKING, PLACING, EMERGENCY, ERROR |
| pick | IDLE, MOVING, PICKING, HOLDING, PLACING, EMERGENCY, ERROR |
| place | IDLE, STANDING, MOVING, PICKING, EMERGENCY, ERROR |

## Deployment

### systemd Services

**Workstation**:

| Service | systemd Unit |
|------|-------------|
| TaskManger | `waic-taskmanger` |
| Navigation (simulation) | `waic-navigation` |
| goal_bridge | `waic-goal-bridge` |
| FoundationPose | `waic-foundationpose` |

**Robot Orin**:

| Service | systemd Unit |
|------|-------------|
| Gateway | `waic-gateway@<domain>` |
| SONIC | `waic-sonic-real@<domain>` |
| ZMQ Bridge | `waic-zmq-bridge@<domain>` |
| Action Bridge | `waic-action-bridge` |
| FAST-LIO | `waic-fastlio` |

### Install systemd Units

```bash
# Workstation
cp systemd/workstation/*.service ~/.config/systemd/user/
# After updating paths as needed:
systemctl --user daemon-reload
systemctl --user enable --now waic-taskmanger

# Orin (copy from the workstation)
scp systemd/orin/*.service unitree@<orin_ip>:~/.config/systemd/user/
ssh unitree@<orin_ip> "systemctl --user daemon-reload"
```

### Startup Order

```text
1. Orin: waic-gateway, waic-fastlio, waic-zmq-bridge, waic-action-bridge
2. Workstation: waic-goal-bridge
3. Workstation: waic-navigation
4. Workstation: waic-foundationpose
5. Workstation: waic-taskmanger
6. Manual: npm run dev (frontend)
```

## New Workstation Setup Checklist

Follow these steps when setting up another workstation:

- [ ] Clone the repository and check out `main`
- [ ] Install ROS2 Humble, conda, and Node.js ≥ 18
- [ ] `colcon build --symlink-install`
- [ ] Create `.venv` and run `pip install -r requirements.txt`
- [ ] Create the `foundationpose` conda environment with PyTorch and FoundationPose dependencies
- [ ] Edit `robots.json` with the actual robot IP addresses and domains
- [ ] Generate `active_robot.env`
- [ ] Edit `scene_coords.json` with the actual scene coordinates
- [ ] `cd phi_robot_fronted && npm install && npm run build`
- [ ] Install the systemd units in `~/.config/systemd/user/` and update paths as needed
- [ ] Generate the `id_ed25519_waic` SSH key for access to the Orin
- [ ] Start the services

## Troubleshooting

**Q: Startup fails with `ModuleNotFoundError: No module named 'rclpy'`.**

Source the ROS2 environment first: `source /opt/ros/humble/setup.bash`.

**Q: The FP node crashes with `AssertionError: matched_object_id must be in [-128, 127]`.**

The tracker ID has overflowed. Change `matched_object_id` and `box_object_ids`
in `SelectTarget.srv` from `int8` to `int32`.

**Q: FP does not receive `/G1Env/env_state_act` data.**

The QoS settings are incompatible. Use `best_effort` for the FP subscription to
match the SONIC publisher's QoS.

**Q: The `frame_id` in `/foundationpose/pose_results` alternates between pelvis and torso_link.**

Multiple FP node instances are running. Stop the old instances with
`pkill -f foundationpose_node`, then restart the node.

**Q: Clicking RUN on the main page does not start execution.**

This is intentional in the grid workflow: the main page only loads the plan into
`StepDebugController`. The operator controls execution from `debug.html`.

**Q: The robot does not respond.**

1. Verify that the DDS domains match: `echo $ROS_DOMAIN_ID`.
2. Check that both systems are on the same local network: `ros2 node list`.
3. Check the ZMQ bridge status: `systemctl --user status waic-zmq-bridge@<domain>`.

**Q: The robot cannot walk after lifting a box.**

Ensure that `/locomotion_mode` is set to `2` (carrying mode). Gateway requires
mode 2 to start walking from the `carry_wait_walk` state.

## Git Workflow

```text
main
  └── <feature-branch> → development → real-robot validation → merge
```

| Rule | Description |
|------|------|
| Commit prefixes | `feat:` / `fix:` / `chore:` / `docs:` |
| Do not force-push to protected branches | |
| Changes to `robot_state.py` require two-person review | SafetyStateMachine affects physical robot safety |
