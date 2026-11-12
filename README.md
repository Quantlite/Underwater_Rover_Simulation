# Coratia ROV Telemetry Ground Station

A native desktop application for the Coratia Technologies **ROV Telemetry Ground Station** technical assessment. It simulates a small underwater ROV and provides operator controls, live telemetry, a depth plot, an OpenStreetMap tile view, safety handling, and CSV recording.

> **Simulation only.** This project does not connect to, command, or validate a physical vehicle. It is a desktop assessment implementation with a deliberately simplified vehicle model.

## 1. Assessment coverage

| Assessment item | Where it is implemented |
|---|---|
| Forward, sideways, vertical movement and turning | Four-axis dynamics in `rov_ground_station/simulator.py` |
| Drag and short thruster-response delay | Linear drag, 120 ms command delay, and first-order thrust response |
| Armed state, mode, roll, pitch, yaw, depth and battery | Live dashboard metrics and vehicle status card |
| Arm / Disarm and Manual / Depth Hold / Heading Hold | Sidebar vehicle and mode controls |
| Keyboard control in Manual mode | W/S, A/D, R/F and Q/E keyboard mapping |
| Operator-set target depth and heading | Validated target inputs and PD hold controllers |
| Zero thrust when disarmed | `ROVSimulator.disarm()` immediately clears all four output channels |
| One-second Manual input watchdog | Hard zero-thrust cut after 1.0 s with no operator input |
| Live depth-versus-time plot | Custom Tkinter Canvas plot, refreshed at 5 Hz |
| OpenStreetMap inside the desktop application | Embedded tile canvas; loads the visible OSM viewport and caches tiles locally |
| Responsive UI | 50 Hz Tk event loop; map downloads and CSV file writes run on worker threads |

### Additional features

- **Emergency Stop:** one action disarms the vehicle and clears outputs immediately.
- **Simulated water current:** adds a configurable environmental drift vector to vehicle position.
- **CSV telemetry logging:** writes timestamped state, setpoints, thruster outputs, watchdog state, and current state to `logs/` on a background writer thread.
- **Flight event log:** records arming, disarming, mode changes, target changes, watchdog action, and logging state.
- **Attitude indicator and thruster monitor:** visual roll/pitch/heading display plus four signed control-output bars.
- **Unit tests:** checks safety, delayed response, hold modes, heading wraparound, bounds, current behaviour, and CSV logging.

## 2. Requirements

- Python 3.10 or later
- Tkinter (usually included with Python; on Ubuntu install `python3-tk` if it is missing)
- Pillow for decoding and displaying OpenStreetMap tile images
- An internet connection for live OpenStreetMap tiles. The map panel displays a clearly labelled offline grid preview when tile access is unavailable. Simulated telemetry and controls work without an internet connection.

No ROS, MAVLink, flight-controller firmware, browser, web server, or physical hardware is required.

## 3. Setup and run (Ubuntu 22.04 / Linux)

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-tk

cd coratia_rov_ground_station
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt

python -m rov_ground_station
```

Alternatively, run:

```bash
./run.sh
```

The GUI opens in a desktop window. It starts **disarmed**. Click **ARM VEHICLE** before sending manual commands. Use **E-STOP** or **DISARM VEHICLE** to cut thrust.

To capture the scripted interface walkthrough from an active Linux X11 desktop (requires `ffmpeg`):

```bash
sudo apt install -y ffmpeg x11-utils
./record_demo.sh Coratia_ROV_Walkthrough.mp4
```

Review the recording before submission. In particular, confirm the map status reads **OSM LIVE**; this build environment had no outbound network access, so its generated sample recording uses the labelled offline preview.

### Keyboard controls

| Key | Command |
|---|---|
| W / S | Forward / reverse (surge) |
| A / D | Left / right (sway) |
| R / F | Descend / ascend (heave) |
| Q / E | Turn counter-clockwise / clockwise (yaw) |
| Space | Neutral manual command |

Keyboard input is accepted only while the vehicle is armed and in **Manual** mode. Release the key to return the associated command to zero. The Manual watchdog cuts all thrust if no operator input is received for 1.0 second.

### Hold modes

1. Select **Depth Hold**, enter a target in metres (0 to 50 m), and press **SET DEPTH**.
2. Select **Heading Hold**, enter a target in degrees, and press **SET HEADING**. Values are normalized to 0–359.9 degrees.
3. The hold controllers use bounded proportional-derivative (PD) commands. Their outputs are clamped to the interval [-1, +1].

Changing into a hold mode initially captures the current depth or heading; applying a target after the mode switch then updates the setpoint.

## 4. Simulation model

The model is purposefully compact and inspectable rather than a full hydrodynamics package.

### Control channels

The operator command is a four-element vector:

`u = [surge, sway, heave, yaw]`, where each element is clamped to [-1, +1].

### Thruster response

Commands are placed in a time history and applied after **120 ms**. The simulated output then follows the delayed command using a first-order response with a **0.16 s** time constant.

### Translational drag

The simplified acceleration equations are:

```text
a_surge = 1.35 * thrust_surge - 0.85 * surge_velocity
a_sway  = 1.10 * thrust_sway  - 0.95 * sway_velocity
a_heave = 0.95 * thrust_heave - 1.10 * heave_velocity
```

Depth is positive down. Heading is degrees clockwise from north. The local north/east position is computed from surge, sway, and the simulated current vector.

### Hold controllers

Depth Hold uses:

```text
heave_command = clamp(0.75 * (target_depth - depth) - 0.85 * heave_velocity)
```

Heading Hold uses the shortest signed heading error in [-180, 180):

```text
yaw_command = clamp(0.035 * heading_error - 0.10 * yaw_rate)
```

These gains are demonstration values, not tuned vehicle-control gains.

### Safety behaviour

- The initial state is disarmed.
- While disarmed, all four thrust outputs are forced to zero.
- Disarming and E-STOP clear the outputs synchronously.
- In Manual mode, if no input arrives for 1.0 second, the watchdog clears the manual command and hard-cuts the outputs to zero. This is intentionally a hard cut rather than a slow ramp-down.
- The watchdog is not applied to Depth Hold or Heading Hold, because those modes continue to calculate their own bounded setpoint commands.

## 5. OpenStreetMap behaviour and attribution

The map panel requests only the tiles needed for the visible viewport from `tile.openstreetmap.org`, caches successful responses locally for at least seven days before refreshing, and displays `© OpenStreetMap contributors` at the bottom-right of the map. The tile URL can be switched with the `ROV_OSM_TILE_URL` environment variable (keep the `{z}`, `{x}`, and `{y}` placeholders). Successful responses are cached under:

`~/.cache/coratia_rov_ground_station/osm/`

Map requests run on background worker threads and use an identifying User-Agent. The application does not prefetch an area or download tiles in bulk. The ROV marker is a **simulated position** around the sample demonstration coordinate 10.8000° N, 74.4000° E; it is not GPS data. If the machine has no network access, the application shows an offline grid preview and keeps the rest of the ground station operational.

Map data and tiles: **© OpenStreetMap contributors**. See <https://www.openstreetmap.org/copyright> for attribution and usage information. The default tile integration follows the published tile policy: application-identifying User-Agent, on-demand viewport requests only, visible attribution, and a seven-day local cache. OSM tile availability is best-effort and requires internet access. See <https://operations.osmfoundation.org/policies/tiles/>.

## 6. CSV logging

Click **START CSV LOG**. A timestamped file is created in `logs/`, for example:

`logs/rov_telemetry_20261111_143000.csv`

The logger runs on a dedicated writer thread so file I/O does not block the interface. It records at approximately 10 Hz while the simulation itself advances at 50 Hz.

## 7. Tests

Install pytest and run:

```bash
python -m pip install pytest
python -m pytest -q
```

The test suite covers:

- Disarmed thrust lock and immediate disarm
- Thruster command delay and response
- One-second Manual watchdog
- Depth Hold response direction
- Heading Hold shortest-turn direction and wraparound
- Target validation
- Current enable/disable behaviour
- CSV file creation and telemetry fields

## 8. Project structure

```text
coratia_rov_ground_station/
├── main.py
├── run.sh
├── requirements.txt
├── README.md
├── INTERVIEW_NOTES.md
├── VALIDATION.md
├── SUBMISSION_CHECKLIST.md
├── rov_ground_station/
│   ├── __init__.py
│   ├── __main__.py
│   ├── app.py          # Native Tkinter interface and event loop
│   ├── logger.py       # Background CSV writer
│   ├── map_view.py     # OSM tile canvas, cache, and offline fallback
│   └── simulator.py    # Dynamics, controllers, and safety logic
└── tests/
    ├── test_logger.py
    └── test_simulator.py
```

## 9. AI-tool disclosure

**Tool used:** ChatGPT (OpenAI).

**How it was used:** to help plan the module boundaries, draft parts of the Python implementation and README, generate an initial set of unit tests, and review/debug edge cases in the safety and mode-switching logic. The project is local source code, not an AI-hosted application. The candidate should run the application, read the implementation, verify the behaviour, and be prepared to explain and modify every module during the interview.

---

**Submission note:** send the source ZIP and walkthrough video by replying to the original Coratia email thread, before **Friday, 13 November 2026, 1:30 PM IST**. The assessment also asks candidates to confirm participation within 24 hours of receiving the invitation.
