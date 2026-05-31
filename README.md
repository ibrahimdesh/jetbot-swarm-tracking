# Cooperative Target Tracking with a JetBot Swarm

A four-robot [Waveshare JetBot](https://www.waveshare.com/wiki/JetBot_AI_Kit) swarm that cooperatively locates and tracks a red cube in a GPS-denied indoor environment, using only on-board monocular cameras and a shared Wi-Fi link. Three coordination strategies are implemented and compared: **independent tracking**, **locked convoy**, and a **social-attraction** controller with PSO-guided search arcs.

Accompanies the project report *Cooperative Target Tracking with a JetBot Swarm: Implementation of three cooperation schemes for finding and tracking a target* (BTH, ET2627 Robotics, 2026).

## What's in here

| Path                          | What it is                                                       |
| ----------------------------- | ---------------------------------------------------------------- |
| `server.ipynb`                | Runs on each JetBot. Camera capture, HSV detection, HTTP server. |
| `social-attractor.ipynb`      | Runs on the laptop. Polls all bots, fuses state, sends commands. |
| `report/`                     | LaTeX source and compiled PDF of the project report.             |
| `figures/`                    | Diagrams and photographs referenced by the report.               |

## How it works (one paragraph)

Each JetBot runs an on-board HTTP server that captures a 3000×2000 frame, crops the workspace, downsamples to 600 px wide, performs **one** BGR→HSV conversion shared across five colour masks (red cube + four peer wraps), scores contours by area / aspect / solidity, and exposes a compact `/state` payload at ~14 Hz. A laptop polls all four bots in parallel at 10 Hz, maintains a smoothed shared world model, assigns each bot a role (leader, tracker, follower, searcher), and issues `/command` messages with a 500 ms watchdog TTL. Bots that see the cube approach it; bots that don't are pulled toward informed peers (social attraction); bots that see neither follow PSO-guided forward arcs.

## Quick start

### On each JetBot

1. Open `server.ipynb` on the JetBot's Jupyter session.
2. In §2.0, set `my_own_color` to that bot's wrap colour (`'blue'`, `'orange'`, `'yellow'`, or `'purple'`).
3. Run §1 → §5. The smoke test should report ≥ 10 Hz detection.
4. First-time only: run §6.1 / §6.2 to calibrate cube and peer distances; §7 to recalibrate HSV under your lighting. All results persist to `/tmp/jetbot_calibration.json`.
5. Run §8 → §10 to start the detection thread and HTTP server. Note the bot's IP.

### On the laptop

1. Open `social-attractor.ipynb`.
2. In §1, edit `bot_registry` with the IPs and assigned colours of your bots. Start with two bots active and uncomment the rest after a successful two-bot run.
3. Run §1 → §4. The self-test confirms each bot is reachable and reports its detection rate.
4. Run §5 → §7 to load the controller and start the debug view (one annotated camera pane per bot).
5. Place the red cube on the floor, position the bots roughly facing it, and run §8.1 to start tracking. §8.2 stops them.

## Endpoints (server side)

| Method | Path                  | Purpose                                                      |
| ------ | --------------------- | ------------------------------------------------------------ |
| GET    | `/state`              | JSON snapshot: cube visibility, bearing, distance, peers.    |
| GET    | `/camera`             | Full uncropped frame (for verifying the crop bounds).        |
| GET    | `/camera_annotated`   | Cropped frame with cube and peer bounding boxes drawn.       |
| GET    | `/health`             | Detection thread status and measured loop rate.              |
| GET    | `/detection_debug`    | Per-colour rejection reasons for diagnostics.                |
| POST   | `/command`            | `{left_motor_speed, right_motor_speed, command_ttl_milliseconds}` |

## Hardware

- 4× Waveshare JetBot kits (NVIDIA Jetson Nano, CSI camera, differential drive)
- Full-chassis coloured paper wraps: orange, blue, yellow, purple
- Red cardboard cube (~10 cm side)
- Shared 2.4 GHz Wi-Fi network with reachable IPs
- One laptop for the central brain

The full-chassis wraps were the single most important hardware decision in the project — small antenna-mounted markers proved unreliable beyond ~1 m. See report §4.3.1 for the rationale.

## Coordination schemes

- **Scheme A — Independent tracking.** Each bot tracks the cube alone; peers are pure obstacles.
- **Scheme B — Locked convoy.** A leader is elected on first cube sighting; the rest form a 0.20 m-spaced chain behind it.
- **Scheme C — Social attraction (preferred).** Informed bots approach the cube; uninformed bots are pulled toward informed peers; a continuous repulsion term enforces separation; bots that see neither cube nor useful peer run PSO-guided forward arcs.

All three live in `social-attractor.ipynb`; the active scheme is whichever the controller selects per-tick from the role-assignment tree in §5.

## Known limitations

- **Illumination.** Direct sunlight on the floor produces warm reflections that the red mask cannot always reject. Diffuse light is markedly more reliable.
- **Wheel envelope.** Repulsion is tuned against the chassis silhouette; tightly packed configurations can produce wheel-to-wheel contact even when the chassis-to-chassis distance is safe.
- **Workspace size.** The convoy needs corridor space to unfold; small arenas favour Scheme C.

See report §4.4 for the full discussion.


## Citation

If you use this work, please cite the project report (details in `report/`).
