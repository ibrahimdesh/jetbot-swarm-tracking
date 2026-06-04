# Cooperative Target Tracking with a JetBot Swarm

A four-robot [Waveshare JetBot](https://www.waveshare.com/wiki/JetBot_AI_Kit) swarm that cooperatively locates and tracks a red cube in a GPS-denied indoor environment, using only on-board monocular cameras and a shared Wi-Fi link.

This repository contains the implementations of the **three coordination schemes** developed and compared in the accompanying project report (BTH, ET2627 Robotics, 2026). The schemes are presented here in the same order as the report, from highest individual autonomy to the cooperative final solution.

## 📖 Full report [here!](https://drive.google.com/file/d/1wsPWoI89N6VdKXj_qOBq7VKkvid7m6oV/view?usp=share_link)

---

## The problem

Four JetBots must locate a red cube placed on the floor and converge on it without colliding, with no external localisation: no GPS, no motion capture, no shared map. Every spatial decision is derived from the on-board camera alone — distance from the apparent size of detected blobs, bearing from their horizontal position in the frame. The target may be moved mid-run, teammates are constantly in motion, and ambient lighting varies.

## System architecture

Perception and coordination are separated across two processes:

- **On each JetBot:** an HTTP server captures frames, runs HSV-based detection for the red cube and four coloured peer wraps, and exposes a compact `/state` payload at ~14 Hz.
- **On a laptop:** a controller (the "brain") polls all four bots in parallel at 10 Hz, fuses their reports into a shared world model, assigns each bot a role, and returns motor commands with a 500 ms watchdog TTL.

The on-board perception pipeline is shared across all three schemes; what changes between schemes is the controller running on the laptop.

## The three schemes

### `scheme-a-independent/` — Independent tracking with avoidance

The literal reading of the brief. Each bot tracks the cube under its own perception and treats peers purely as obstacles to avoid. Reliable but information-isolated: a bot whose starting orientation doesn't include the cube spends a large fraction of each run rotating in place to search, even when teammates are already approaching the cube and could in principle have led it there.

### `scheme-b-convoy/` — Locked convoy chain

The team is constrained into a train-like file. The first bot to hold the cube in view becomes the leader; the rest are ordered behind it by distance and assigned to follow the bot one slot ahead at a fixed 0.20 m spacing. The most predictable scheme and the easiest to describe to a human observer, but the rigid formation ignores the geometry of the environment.

### `scheme-c-social-attraction/` — Social attraction *(final solution)*

The preferred scheme and the one most faithful to the project brief. A bot that sees the cube approaches it directly; a bot that doesn't is pulled toward a teammate that does, while a continuous repulsion term keeps every pair clear. Bots that see neither cube nor a useful peer run **PSO-guided forward arcs** rather than spinning in place, so search remains directed rather than random. The result is a vision-only realisation of Reynolds-style cohesion and separation, with cohesion asymmetrically activated only where it carries information.

## Repository layout

```
.
├── scheme-a-independent/
│   ├── server.ipynb       # runs on each JetBot
│   └── client.ipynb       # runs on the laptop
├── scheme-b-convoy/
│   ├── server.ipynb
│   └── client.ipynb
├── scheme-c-social-attraction/
│   ├── server.ipynb
│   └── client.ipynb
└── README.md
```

Each folder is self-contained — open the pair of notebooks inside it to run that scheme.

## Quick start

### 1. On each JetBot

1. Open `server.ipynb` from the scheme folder you want to run.
2. In §2.0, set `my_own_color` to that bot's wrap colour (`'blue'`, `'orange'`, `'yellow'`, or `'purple'`).
3. Run the cells through to the HTTP server startup. The smoke test should report ≥ 10 Hz detection.
4. First-time only: run the calibration cells to set the cube and peer distance constants and to fit HSV ranges under your lighting. Results persist to `/tmp/jetbot_calibration.json`.
5. Note the bot's IP — you'll need it on the laptop.

### 2. On the laptop

1. Open `client.ipynb` from the same scheme folder.
2. In §1, edit `bot_registry` with the IPs and assigned colours of your bots. Start with two bots active for first runs and uncomment the rest after a successful two-bot trial.
3. Run the self-test — it confirms every bot is reachable and reports its detection rate.
4. Start the debug view (one annotated camera pane per bot).
5. Place the red cube, position the bots roughly facing it, and start the controller. A stop cell is provided.

## Hardware

- 4× Waveshare JetBot kits (NVIDIA Jetson Nano, CSI camera, differential drive)
- Full-chassis coloured paper wraps: orange, blue, yellow, purple
- Red cardboard cube (~10 cm side)
- Shared 2.4 GHz Wi-Fi network with reachable IPs
- One laptop for the central brain

The full-chassis wraps were the single most important hardware decision in the project — small antenna-mounted markers proved unreliable beyond about a metre. The report discusses this in detail.

## HTTP endpoints

All three servers expose the same interface:

| Method | Path                  | Purpose                                                           |
| ------ | --------------------- | ----------------------------------------------------------------- |
| GET    | `/state`              | JSON snapshot: cube visibility, bearing, distance, visible peers. |
| GET    | `/camera`             | Full uncropped frame (for verifying the crop bounds).             |
| GET    | `/camera_annotated`   | Cropped frame with cube and peer bounding boxes drawn.            |
| GET    | `/health`             | Detection thread status and measured loop rate.                   |
| GET    | `/detection_debug`    | Per-colour rejection reasons for diagnostics.                     |
| POST   | `/command`            | `{left_motor_speed, right_motor_speed, command_ttl_milliseconds}` |

## Known limitations

- **Illumination.** Direct sunlight on the floor produces warm reflections that the red mask cannot always reject. Diffuse light is markedly more reliable.
- **Wheel envelope.** Repulsion is tuned against the chassis silhouette; tightly packed configurations can produce wheel-to-wheel contact even when the chassis-to-chassis distance is safe.
- **Workspace size.** The convoy needs corridor space to unfold; small arenas favour the social-attraction scheme.

The report discusses these in detail and points to a learned detector as the natural next step for the perception side.

## License

Released under the [MIT License](LICENSE).



