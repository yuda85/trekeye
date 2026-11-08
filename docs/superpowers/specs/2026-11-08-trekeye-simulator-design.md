# TRKEYE Simulator — Design Spec

- **Date:** 2026-11-08
- **Status:** Approved in brainstorming, pending written-spec review
- **Source document:** *RACELOGIC VBOX Integration with TRKEYE — Technical paper, 5 November 2026* (referred to below as "the paper")

## 1. Purpose and context

TRKEYE will display an adaptive racing line, braking and turn-in/exit cues inside a custom Eye-Lights helmet. The paper describes the full in-car installation: VBOX 4 Dynamics with dual antennas and IMU05-S, VBOX Video HD2 with Sigma RTK, CAN Gateway, ADC03 with hydraulic pressure and pedal force, eight infrared tyre/rotor sensors and four suspension-travel sensors. Target car: Porsche 992 GT3 RS. Target circuit: Yas Marina.

Project sequence: today there is only the paper. Next comes access to the car and the full hardware configuration, from which real logs will be produced. The Eye-Lights kit comes last.

Milestone 1 is **track-day readiness**: when the car arrives, the first session must produce usable, decodable, time-aligned data. This simulator is built in parallel to support that milestone. It is a configurable service that loads tracks, simulates the car, and emits data that is **structurally identical** to what the installed sensors will produce: same buses, same message layouts, same rates, realistic latency, jitter, noise, clock drift, dropouts and quality degradation.

## 2. Goals and non-goals

### Goals (v1)

1. Simulate laps of a loaded track with a dynamic single-track vehicle model and a configurable driver.
2. Emulate every sensor of the paper's installed configuration as a separate plugin with realistic timing and error models.
3. Publish live data per physical bus and write logger-style files, from one core, in **batch** (faster than real time) and **real-time** modes.
4. Publish ground truth separately so that decoders and time alignment can be validated quantitatively.
5. Inject faults on a timeline or live (RTK float/loss, correction dropouts, bus dropouts, clock steps).
6. Provide a web dashboard with a realistic procedural 3D track, a driver-eye view with a racing-line ribbon, traces, sensor health and simulation control.
7. Share the track model, data contracts and decoders with the rest of the TRKEYE application, so that switching from simulator to real car changes the data source, not the code.
8. Meet the data-rate and latency budgets in §5 with measured evidence.

### Non-goals (v1)

- Google Photorealistic 3D Tiles surroundings (phase 2).
- Synthetic camera video, cabin IR tracking cameras, helmet pose and head-motion model (phase 2).
- Two-track / full multibody vehicle physics (phase 2; the vehicle interface is pluggable).
- Racing-line optimisation beyond a minimum-curvature line.
- Heavy brain model fitting, reaction brain guidance and Eye-Lights rendering.
- Car-to-paddock wireless link (requirements recorded in §5.9 only).
- Docker, authentication, multi-machine deployment.

## 3. Decisions

| Topic | Decision | Reason |
|---|---|---|
| Repository | One monorepo: uv workspace for Python, pnpm workspace for TypeScript, shared `contracts/` with code generation for both | One version of every wire format; atomic changes across services and dashboard |
| Language | Python 3.12 for services; TypeScript for the dashboard | Fast development, strong data/scientific ecosystem; Windows laptop at the track |
| Vehicle physics | Dynamic single-track (bicycle) with simplified Pacejka, quasi-static load transfer, lumped thermal models; pluggable `VehicleModel` | Produces genuine slip, yaw, driver inputs and response at modest cost |
| Output modes | One core, two modes: batch (as fast as possible, writes files) and real-time (wall-clock paced live streams) | Batch for test data and CI; real-time for live consumers |
| Sensor separation | Each sensor is a separate **plugin** inside one `sim` process with one deterministic scheduler; **service separation happens at the physical-bus level** | Avoids IPC fan-out of high-rate ground truth and distributed lockstep; keeps determinism; matches reality, where consumers see buses and logger files, not sensors |
| Communication | Four planes with different transports (§5): in-process sinks for batch, ZeroMQ PUB/SUB with batched columnar frames for the live data plane, request/reply for control, binary WebSocket with per-stream QoS to the browser; shared memory reserved for large payloads | Each plane has different rate, latency and reliability needs; per-frame messaging cannot meet batch throughput in Python |
| Transport choice gate | ZeroMQ is the v1 default behind a `Transport` interface; Zenoh is benchmarked against it in phase 1 and replaces it only if it meets the budgets and wins on measured results | Decide on evidence from the target laptop |
| Dashboard | Web: FastAPI + WebSocket gateway, three.js front end | Viewable from any pit laptop or tablet; reusable for real car data; path to 3D/AR preview |
| 3D realism | Procedural track mesh built from the shared track model, PBR materials, HDRI lighting; real surroundings in phase 2 | Visuals always match the simulated geometry; no external API dependency in v1 |
| Track data | Loader adapters; TUM racetrack-database Yas Marina as the first source | Available now; known limitations recorded in §14 |

## 4. Architecture

### 4.1 Services

```
            ┌──────────────────────────── sim (one process) ─────────────────────────────┐
scenario ──►│ track model ─► vehicle model ◄─ driver model                                 │
  YAML      │                    │ truth @1 kHz (ring buffer)                              │
            │                    ▼                                                         │
            │  sensor plugins (VBOX 4, IMU, NTRIP, Sigma, HD2, Gateway, vehicle CAN,      │
            │                  ADC03, IR tyre ×4, IR rotor ×4, suspension ×4)             │
            │                    │ emissions (topic, frame, t_deliver)                     │
            │                    ▼                                                         │
            │  discrete-event scheduler ─► batcher ─► sinks                               │
            │                                          ├─ in-process (batch: files, tests)│
            │                                          └─ transport (real-time: ZeroMQ)   │
            └──────────┬──────────────────────┬──────────────────────┬───────────────────┘
                       │ bus.*                │ log.*                │ sim.truth / sim.status
                       ▼                      ▼                      ▼
                   live data plane (ZeroMQ PUB/SUB, batched columnar frames)
                       │                      │                      │
                  gateway (FastAPI,      logger-emu              (future: reaction brain,
                  runs ingest decoders,  (writes .vbo,            ingest service,
                  WebSocket → dashboard) CAN archives)            recorders)
```

| Service | Responsibility |
|---|---|
| `sim` | Scenario, track, vehicle, driver, sensor plugins, scheduler, batching, dispatch, fault injection |
| `logger-emu` | Writes files in the formats the real loggers produce. In batch mode it runs as an in-process sink; in real-time mode as a subscriber |
| `ingest` | Decoders and time alignment (library used by the gateway in v1; standalone service later) |
| `gateway` | REST control API, decoding of live buses, decimation, WebSocket streams to the dashboard |
| `dashboard` | Browser UI: 3D view, minimap, traces, sensor health, control |

### 4.2 Monorepo layout

```
trekeye/
  pyproject.toml                 uv workspace root (members: packages/*)
  pnpm-workspace.yaml            pnpm workspace root (members: apps/*, packages-ts/*)
  contracts/
    schemas/                     wire-format definitions (YAML): batch dtypes, truth record,
                                 topic catalogue with rate/QoS, UI stream layouts
    dbc/                         CAN databases (public-spec and *.PLACEHOLDER.dbc)
    vbo/                         VBO channel specifications
  packages/                      Python (uv)
    trekeye-contracts/           generated: numpy dtypes, pydantic models, topic constants
    trekeye-core/                sim-clock and time types, coordinate frames (ENU/WGS84),
                                 track model, Transport interface + backends
    trekeye-sim/                 scenario loader, vehicle, driver, sensor plugins, scheduler
    trekeye-logger-emu/          logger-format writers
    trekeye-ingest/              decoders, time alignment
    trekeye-gateway/             FastAPI app, WebSocket streaming, decimation
  packages-ts/                   TypeScript (pnpm)
    contracts-ts/                generated: DataView readers and TS types for UI streams
  apps/dashboard/                Vite + three.js + uPlot
  tracks/  cars/  scenarios/     data
  tools/
    codegen.py                   contracts/schemas → packages/trekeye-contracts, packages-ts/contracts-ts
    run.py                       launcher that starts the services together
    bench/                       transport and pipeline benchmarks (§5.8)
  docs/superpowers/specs/        design specs
  docs/prompts/                  design prompts
```

`contracts/schemas` is the single source of truth. `tools/codegen.py` generates the Python and TypeScript code; a CI check fails if generated code is stale.

### 4.3 Realism principle

Consumers receive only what they would receive from real hardware: raw frames and a receipt timestamp from their own clock. Measurement time appears only where the real device embeds it (for example GPS time inside a VBOX payload). True times and true state exist only on `sim.truth`.

### 4.4 Clocks

The simulator owns true time (integer nanoseconds). Every device (VBOX 4, Sigma, HD2, ADC03, each IR sensor, the receiving computer) has its own clock model: initial offset, drift in ppm and optional step faults. Devices sample on their own clock, so drift appears as rate deviation. This reproduces the alignment problem the paper describes, and `ingest` must solve it.

## 5. Communication architecture

### 5.1 Data-rate budget

Frame counts are estimates for the emulated channel set and are verified against the RACELOGIC databases in phase 3.

| Topic | Source | Frames or records / s (real time) | Size | Transport latency budget | Delivery semantics |
|---|---|---|---|---|---|
| `bus.vbox4` | 100 Hz × ~16 IDs | ~1,600 | 8 B data | ≤ 1 ms p99 | Lossless, ordered |
| `bus.vehicle` | 10–100 Hz per ID | ~1,000 | 8 B | ≤ 1 ms p99 | Lossless, ordered |
| `bus.thermal` | 8 sensors × 16 points, 4 points per frame | up to ~3,200 at 100 Hz | 8 B | ≤ 5 ms p99 | Lossless, ordered |
| `bus.module` | ADC03 polled | ~400 | 8 B | ≤ 1 ms p99 | Lossless, ordered |
| `link.sigma_hd2` | 10 Hz | ~20 | small | ≤ 5 ms p99 | Lossless, ordered |
| `log.vbox4` | 100 Hz main + 4 × 500 Hz analogue | ~2,100 samples | small | none (recording) | Lossless |
| `log.hd2` | per channel | ~200 | small | none (recording) | Lossless |
| `sim.truth` | 1 kHz × ~120 values | 1,000 | ~1 KB | ≤ 10 ms | Lossless to recorders; latest-value to UI |
| `sim.status` | 10 Hz | 10 | small | ≤ 50 ms | Latest value |
| `sim.control` | events | rare | small | ≤ 50 ms round trip | Reliable request/reply |

Totals: about 6,000–8,000 bus frames per second plus ~1 MB/s of truth in real time. Batch mode at the 20× target multiplies this to about 130,000–160,000 frames per second.

The `bus.thermal` estimate also drives the CAN model: a classic 8-byte frame is roughly 111–135 bits including stuffing, so 1 Mbit/s carries at most about 7,400–9,000 frames per second, and 3,200 frames per second is about 35–45 % bus load.

### 5.2 Consequence: batch, never send per frame

Python messaging costs on the order of 10–20 µs per message for serialisation and the socket call. At batch-mode rates that is more than one CPU-second per simulated second. Therefore:

- **Frames are never sent one per message.** The batcher packs frames per topic into columnar batches.
- Each frame keeps its own `rx_ns`, so batching never changes timestamps or ordering.
- **Batch mode** uses in-process sinks only (file writers, test collectors). No sockets are involved, which leaves the full CPU for simulation.
- **Real-time mode** flushes a topic's batch when either the flush window elapses (default 1 ms; configurable per topic) or the batch reaches its size limit. With the 1 ms window, added latency is ≤ 1 ms, which is small against the 10 ms VBOX 4 period. Bursty sources such as VBOX 4 (all IDs per 100 Hz cycle) produce about 100 batches per second per bus, so the live data plane carries hundreds of messages per second rather than thousands.

### 5.3 The four planes

| Plane | Path | Transport | Encoding | Semantics |
|---|---|---|---|---|
| **Batch / in-process** | sim → logger-emu, test collectors | Direct function calls on sink objects | numpy structured arrays | Lossless, deterministic |
| **Live data** | sim → gateway, logger-emu, future reaction brain | ZeroMQ PUB/SUB over `tcp://127.0.0.1` (Windows) or `ipc://` (Linux) | Multipart: header + raw columnar body (§5.4) | Per-topic: lossless with sequence numbers, or latest-value |
| **Control** | dashboard → gateway → sim | Browser → gateway: REST/JSON. Gateway → sim: ZeroMQ ROUTER/DEALER | JSON validated by pydantic models | Reliable request/reply with timeouts and idempotent commands |
| **UI** | gateway → browser | WebSocket, binary frames | Schema handshake, then typed-array frames (§5.6) | Per-stream QoS (§5.6) |

Large payloads (phase 2 synthetic camera frames: 1080p RGB at 30 fps ≈ 180 MB/s) never travel through sockets. They are written to a shared-memory ring buffer (`multiprocessing.shared_memory`, or iceoryx2 if benchmarked better), and a small ZeroMQ message carries the slot index and timestamp.

### 5.4 Wire format for the live data plane

Each message is two ZeroMQ frames:

1. **Header** (msgpack, < 100 B): `topic`, `schema_version`, `seq` (per-topic sequence number), `count`, `t_send_ns` (sender wall clock, for transport-latency monitoring).
2. **Body**: raw bytes of a numpy structured array, sent zero-copy and decoded with `numpy.frombuffer` (no per-frame parsing).

Frame dtypes are defined in `contracts/schemas` and generated for every language:

| dtype | Fields | Size |
|---|---|---|
| `can_classic` | `rx_ns: i8`, `id: u4`, `flags: u1` (ext, err), `dlc: u1`, `pad: u2`, `data: u1[8]` | 24 B |
| `can_fd` | same with `data: u1[64]` | 80 B |
| `log_sample` | `t_dev_ns: i8`, `channel: u2`, `pad: u2`, `value: f8` | 24 B |
| `truth` | `t_true_ns: i8` + ~120 `f8` state fields | ~1 KB |

The layout is language-neutral (fixed little-endian binary, documented in contracts), so a future C++ or Rust reaction brain reads it without Python.

### 5.5 Delivery semantics on the live data plane

- **Lossless topics** (`bus.*`, `log.*`, `sim.truth` to recorders): the send high-water mark is sized for at least 2 s of data. Consumers check `seq`, and any gap is reported as an error in sensor health and in recordings. ZeroMQ PUB/SUB can drop at the high-water mark, so this guarantees detection, not delivery. Guaranteed delivery is the job of batch mode with in-process sinks.
- **Latest-value topics** (`sim.status`, UI-bound truth): a separate single-part SUB socket with `ZMQ_CONFLATE`, so a slow consumer always gets the newest value. `ZMQ_CONFLATE` does not support multipart messages, so these topics use a single-part encoding.
- **Slow-joiner handling:** subscribers announce readiness on the control plane before the sim starts real-time dispatch.

### 5.6 UI plane: gateway to browser

- **Handshake:** on connect, the gateway sends a JSON schema message listing streams, channels, units and binary layouts (generated from `contracts/`).
- **Data frames:** a 16-byte header (`stream_id`, `seq`, `count`, `t0`) followed by `Float64Array`/`Float32Array` columns. The browser reads them with typed arrays and does no JSON or msgpack parsing in the hot path.
- **Per-stream QoS:**
  - **Pose (3D view):** latest-value conflation at 60 Hz. The client interpolates between timestamps.
  - **Traces:** decimated with min/max-per-bucket aggregation (M4), so short spikes such as curb strikes at 500 Hz stay visible. When a client falls behind, the oldest trace batch is dropped and a gap marker is sent.
  - **Events and faults:** reliable and ordered. Never dropped.
- **Backpressure:** bounded queue per client; one slow tablet cannot slow the gateway or the other clients.
- **Later:** WebTransport (HTTP/3 datagrams) avoids head-of-line blocking on lossy pit Wi-Fi. The stream layer is written so the socket can be swapped.

### 5.7 Transport options considered

| Option | Strengths | Weaknesses | Decision |
|---|---|---|---|
| **ZeroMQ** (pyzmq) | Mature, brokerless, zero-copy multipart, Windows wheels, µs-level latency on loopback | No discovery, no built-in persistence, PUB drops at high-water mark | **v1 default** for live data and control |
| **Zenoh** | Pub/sub plus query/storage, discovery, shared-memory transport, built for robotics and edge links across networks | Younger Python ecosystem; shared-memory support varies by platform | **Benchmark in phase 1**; strong candidate for the future car-to-paddock link |
| NATS (+ JetStream) | Simple subjects, persistence, good multi-machine story | Requires a server process; adds a hop on the local hot path | Candidate for car-to-paddock store-and-forward |
| DDS / ROS 2 | Rich QoS, tooling (bags, visualisers) | Heavy, complex on Windows, large footprint for one-machine needs | Not used |
| Shared memory (iceoryx2, `multiprocessing.shared_memory`) | True zero-copy, lowest latency | Same machine only; needs a notification channel | Used only for large payloads |
| gRPC streaming | Strong typing, good across languages | Per-message overhead; HTTP/2 head-of-line blocking | Not used on the hot path |
| MQTT, Redis Streams | Simple, widely known | Broker hop; throughput and latency below needs | Not used |
| WebSocket | Universal browser support, simple | TCP head-of-line blocking | **v1 UI plane** |
| WebTransport | Unreliable datagrams, no head-of-line blocking | Python server support less mature | Later, behind the same stream layer |

### 5.8 Communication benchmarks (phase 1 gate)

`tools/bench/` measures on the target Windows laptop, and the results are committed with the hardware description:

1. ZeroMQ vs Zenoh, loopback: p50/p99/max latency and CPU for 100–5,000 messages per second with 0.1–64 KB bodies.
2. End-to-end real-time live data plane at 1× and 2× nominal load from §5.1.
3. Batch pipeline with in-process sinks: simulated seconds per wall-clock second.
4. Gateway to Chrome over WebSocket: 60 Hz pose plus 50 decimated trace channels; frame time in the browser.

Pass criteria:

- Live data plane transport p99 ≤ 1 ms at 2× nominal load.
- Transport CPU ≤ 25 % of one core at 1× nominal load.
- Batch mode ≥ 20× real time with the full sensor set.
- Browser keeps ≥ 60 fps with the UI streams active (≥ 30 fps minimum on weaker hardware).

### 5.9 Future links (requirements only)

- **Real CAN input:** a hardware adapter (python-can with PCAN/Kvaser/SocketCAN) publishes the same `can_classic`/`can_fd` batches on the same topics. Consumers do not change.
- **Reaction brain in the car:** needs ≤ 1 ms p99 from bus receipt to its input queue. It may be written in C++ or Rust and reads the language-neutral wire format.
- **Car to paddock:** intermittent wireless link; requires store-and-forward, resumable transfer of session files and prioritised live summaries. Zenoh and NATS JetStream are the candidates; this is decided when that subsystem is designed.
- **Time synchronisation on the real car:** the reaction computer's clock must be disciplined to GPS time (from the VBOX) or a PTP source; recorded as an open item.

## 6. Simulation core

### 6.1 Track model (`trekeye-core`, shared)

- Parametrised by arc length `s`: `x, y, z`, heading, curvature, banking, grade, `w_left`, `w_right`.
- Curb segments: `s` range, side, profile type and cross-section geometry.
- Geo anchor: origin latitude, longitude and ellipsoidal height plus rotation; conversions ENU ↔ WGS84.
- Loader adapters:
  - TUM racetrack-database CSV (`x_m, y_m, w_tr_right_m, w_tr_left_m`) plus a sidecar YAML with geo anchor and curbs.
  - GeoJSON with left/right edge polylines.
  - Later: RTK survey from the real car.
- Elevation is optional: flat by default, or a synthetic profile.
- The same model is served to the dashboard, which builds the 3D mesh from it.

### 6.2 Vehicle model (`VehicleModel` interface)

v1 implementation: dynamic single-track, fixed-step 1 kHz integration.

- States: planar position, yaw, longitudinal and lateral velocity, yaw rate, wheel rotational speeds, gear, rotor and tyre temperatures.
- Tyres: simplified Pacejka lateral force, slip-ratio longitudinal force, friction-circle coupling.
- Powertrain: engine torque map, gearbox, automatic shift schedule.
- Brakes: force = hydraulic pressure × gain × pad μ(temperature); front/rear circuits; simple ABS.
- Aero: downforce and drag as functions of speed.
- Quasi-static pitch, roll and load transfer. These provide body attitude for the IMU and suspension travel per corner (via motion ratio plus road roughness and curb input).
- Thermal: lumped rotor model (heating from braking power, speed-dependent cooling) and lumped tyre surface model (heating from slip power).
- Parameters in `cars/992_gt3rs.yaml`. Values are approximations marked `placeholder: true` until replaced with measured data.

### 6.3 Driver model (`DriverModel` interface)

- Reference line: minimum-curvature line or offset centreline.
- Speed profile: forward-backward pass over the vehicle's GGV envelope.
- Steering: pure pursuit or Stanley with preview.
- Speed: PI controller mapped to throttle or brake pressure, with brake ramp-rate limits.
- Skill parameters: braking-point noise, lateral line deviation, reaction delay, pressure overshoot.
- Test-card mode: B1/B2 gates with entry speed, target pressure, ramp, hold and release-speed threshold. This reproduces the paper's three-lap calibration programme.

### 6.4 Scheduler, batcher and dispatch

- Discrete-event scheduler on sim time (integer nanoseconds).
- The physics tick writes truth into a ring buffer.
- Each sensor plugin registers sampling events on its device clock. On each sample it:
  1. Interpolates truth at the sample's true time.
  2. Applies its sensor model.
  3. Encodes frames.
  4. Computes delivery time as latency + jitter + CAN serialisation and arbitration delay for that bus's bitrate and load.
- The batcher groups emissions per topic and flushes them to the sinks (§5.2).
- Dispatcher:
  - **Batch:** delivers in delivery-time order to in-process sinks as fast as possible.
  - **Real-time:** paces against the wall clock with a realtime factor (0.5×, 1×, 2×). The pacer runs in a dedicated thread using high-resolution `time.sleep` (Python ≥ 3.11 on Windows) plus a short final spin; asyncio timers are not used for pacing.
- Determinism: one root seed; each plugin gets a child generator via `numpy.random.SeedSequence.spawn`.

### 6.5 Scenario file

```yaml
name: yas-3lap-calibration
seed: 42
mode: batch            # batch | realtime
realtime_factor: 1.0
track: tracks/yas_marina
car: cars/992_gt3rs.yaml
ambient: { air_c: 30, track_c: 42 }
driver:
  model: pursuit_pi
  skill: { brake_point_sigma_m: 2.0, line_sigma_m: 0.15, reaction_ms: 180 }
program:
  - warmup_laps: 1
  - test_cards: [cards/b1_low.yaml, cards/b2_moderate.yaml]
    laps: 3
  - free_laps: 2
sensors: sensors/full_installation.yaml   # per-plugin parameters
transport:
  flush_window_ms: { default: 1.0, bus.thermal: 5.0 }
faults:
  - { at_s: 120, duration_s: 8, kind: rtk_float, device: vbox4 }
  - { at_s: 300, duration_s: 2, kind: bus_dropout, bus: thermal }
  - { at_s: 410, kind: clock_step, device: hd2, step_ms: 15 }
```

### 6.6 Control

Through the control plane (§5.3): start, pause, single step, stop, change realtime factor, inject or clear a fault while running.

## 7. Sensor plugins

### 7.1 Interface

```python
class SensorPlugin(Protocol):
    device: DeviceClock
    def schedule(self, scheduler: Scheduler) -> None: ...
    def sample(self, t_true_ns: int, truth: TruthView) -> list[Emission]: ...

@dataclass(frozen=True)
class Emission:
    topic: str          # "bus.vbox4", "log.vbox4", ...
    frame: Frame        # raw CAN frame or log record
    t_deliver_ns: int   # true time at which the consumer receives it
```

### 7.2 Shared realism layers (composable)

- **Clock:** offset, drift in ppm, step faults.
- **Noise:** Gaussian, bias random walk.
- **Bandwidth:** first-order low-pass at the sensor bandwidth.
- **Quantisation:** from the DBC scaling.
- **Latency:** fixed component plus a jitter distribution.
- **Dropout:** probability and burst length.
- **Quality:** status flags derived from conditions and active faults.

### 7.3 Catalogue

| Plugin | Output | Rate | Key models |
|---|---|---|---|
| VBOX 4 Dynamics + dual antenna | `bus.vbox4`, `log.vbox4` | 100 Hz | Position error by fix type (RTK fixed ≈ 2 cm, float 20–50 cm, standalone ≈ 1.5 m); antenna lever arms; true heading from baseline; fused attitude and slip; fix/filter status |
| IMU05-S | via VBOX 4 | 100 Hz on CAN | ±10 g range, angular rates, mount misalignment, bias |
| Suspension travel ×4 | `log.vbox4` only | 500 Hz | Zero, polarity, motion ratio, sensor bandwidth |
| NTRIP modem | affects VBOX 4 fix | — | Correction age, dropouts |
| Sigma (RLACS337) | `link.sigma_hd2`, Sigma CAN | 10 Hz | Own antenna and lever arm; independent correction link and faults |
| HD2 | `log.hd2` | per channel | .vbo writing; video as frame timestamps only in v1 |
| CAN Gateway (RLVBCGW01) | routes a subset to HD2 | — | Filtering, added latency |
| Vehicle CAN (992) | `bus.vehicle` | per ID | ECU pedal, steering, RPM, wheel speeds ×4, ABS, gear |
| ADC03 | `bus.module`, pass-through on `bus.vbox4` | polled | Front/rear hydraulic pressure (bar), pedal force (N, RLACS282) |
| RLACS272 ×4 (tyre) | `bus.thermal` | per database | 16 points per sensor, emissivity error, aim |
| RLACS314 ×4 (rotor) | `bus.thermal` | per database | 16 points per sensor, emissivity error |

### 7.4 Coordinate and format conventions (from the paper's data contract)

| Interface | Convention |
|---|---|
| Sigma .VBO text | Signed decimal minutes, longitude positive west |
| Sigma CAN 0x301 / 0x302 | Signed 32-bit minutes × 100000, longitude positive west |
| Sigma CAN 0x308 / 0x309 | Signed 48-bit minutes × 10000000, longitude positive east |
| VBOX 4 CAN | VBOX 4 definitions for position, true heading, fused attitude and quality |

### 7.5 Contracts policy

- Where RACELOGIC publishes a specification (VBOX 4 CAN output, Sigma CAN, VBO file structure, module CAN database), the encoder and decoder follow it exactly.
- Where no public specification exists, notably the full Porsche 992 vehicle CAN, the contract file is named `*.PLACEHOLDER.dbc`. A test counts the remaining placeholders and reports them.
- Placeholders are replaced from sniffed traffic once the car is available.

## 8. logger-emu

- Consumes `log.vbox4` and `log.hd2`, plus every `bus.*` topic for CAN archives.
- Writes:
  - HD2 / Sigma `.vbo` text files (decimal minutes, longitude positive west).
  - VBOX 4 dynamics log including 500 Hz analogue channels.
  - CAN archives of each bus in a standard trace format (for example `.asc` or `.blf` via python-can).
- Each output file carries the timestamps of its own logger's clock. A session manifest (sim side only) records the scenario, seed and versions.

## 9. Ingest (v0 within this spec)

- Decoders for every topic in §5.1, using `cantools` with the files in `contracts/dbc`. Decoding works on whole batches (vectorised over the columnar body), not frame by frame.
- Time alignment v0: estimate per-device clock offset and drift from shared observables (GPS time fields, common events such as brake onset visible in several sources) and resample to a common timeline.
- Every decoded channel carries source, units and quality, as the paper requires.

## 10. Gateway

- FastAPI REST:
  - Lists tracks, cars and scenarios.
  - Returns a track package for mesh building.
  - Exposes sim control (forwarded on the control plane).
- Subscribes to the live data plane, runs the `ingest` decoders and streams **decoded** channels. The dashboard therefore shows what the system actually understands, and decoders are exercised continuously against `sim.truth`.
- Implements the UI plane of §5.6.
- Computes sensor health per device: measured rate, latency, jitter, sequence gaps, fix status, data age and estimated clock offset; also transport health per topic (`t_send_ns` to receipt).

## 11. Dashboard

TypeScript, Vite, three.js (WebGL2; WebGPU later), uPlot.

1. **3D view**
   - Procedural mesh from the track model: PBR asphalt with normal maps, red/white curbs with real cross-section geometry, line markings, run-off, sand, grass and walls along the boundaries.
   - Lighting: HDRI environment plus sun with shadows, including a Yas twilight preset.
   - Assets: CC0 textures (Poly Haven / ambientCG) and an openly licensed generic GT car glTF.
   - Cameras: chase, orbit, TV, top and **driver-eye**. The driver-eye camera shows the racing-line ribbon and braking markers on the road: a virtual Eye-Lights preview.
   - Ghost car: true pose next to the measured/decoded pose, so sensor error is visible.
2. **Minimap:** sectors and lap progress.
3. **Traces:** speed, pressures, steering, temperatures and suspension against time or distance, with truth vs measured overlay.
4. **Sensor and transport health:** per-device and per-topic table with status colours.
5. **Control:** scenario, seed, start/pause/step/stop, realtime factor, fault-injection buttons.

## 12. Testing and acceptance criteria

| Area | Test | Criterion |
|---|---|---|
| Contracts | Codegen up to date; generated Python and TS layouts identical | CI fails on stale or mismatched code |
| Track model | Arc length, curvature, ENU↔WGS84 round trip vs `pyproj` | Round-trip error < 1 mm |
| Vehicle | Steady-state cornering, straight-line braking, energy sanity | Braking decel consistent with μ·g and configured brake model; lap time and peak g within plausibility bands set in the car file |
| Sensor layers | Statistical tests on noise σ, drift ppm, rate, dropout rate | Within 5 % of configured values over long runs |
| Encode/decode | Property-based round trip (`hypothesis`) truth → frame → batch → decode | Within one quantisation step |
| Determinism | Same scenario and seed, two batch runs | Identical output hashes |
| DBC contracts | Decoders against published examples; placeholder counter | All public-spec examples pass; placeholder count reported |
| Integration | 3 Yas laps: batch → logger-emu → ingest → aligned session | Alignment error vs truth ≤ 1 ms; every injected fault flagged in quality |
| Communication | Benchmarks of §5.8 | All pass criteria of §5.8 met on the target laptop |
| Real-time pacing | Emission time vs scheduled time | p99 jitter ≤ 1 ms on Windows |
| Sequence gaps | Forced slow subscriber | Gaps detected and reported, publisher unaffected |
| Dashboard | Playwright smoke test | Loads, renders the track, receives a stream |

## 13. Build phases

Each phase ends with something runnable.

1. **Foundations and communication:** monorepo, `contracts/` with codegen, `trekeye-core` (time, frames, track model, TUM loader with geo anchor), `Transport` interface with ZeroMQ backend, scenario schema, **communication benchmarks of §5.8 including Zenoh**. *Exit:* load Yas Marina and print track statistics; ENU/WGS84 tests pass; benchmark report committed and transport decision recorded.
2. **Vehicle, driver, scheduler:** batch mode, truth only. *Exit:* three plausible laps of Yas with truth exported.
3. **Sensors:** realism layers, all v1 plugins, encoders, batcher, logger-emu. *Exit:* a batch run produces bus streams and logger files for the full installation.
4. **Ingest:** decoders, round-trip tests, time alignment v0. *Exit:* integration test from §12 passes.
5. **Gateway and dashboard:** UI plane, 3D track, car, traces, health, control, real-time mode. *Exit:* a live real-time session is viewable in the browser and §5.8 browser criteria pass.
6. **Faults, test cards, driver-eye:** fault timeline and live injection, test-card driver mode, ribbon and braking markers. *Exit:* the three-lap calibration scenario runs with injected faults visible in the dashboard.

## 14. Risks and open questions

- **Yas Marina geometry:** the TUM dataset most likely predates the 2021 layout change (Turns 5–6 chicane removed, Turns 7 and 11–14 revised). The current layout needs another source (GeoJSON traced from imagery, then RTK survey).
- **Geo anchor:** TUM coordinates are local metres. An anchor must be fitted against imagery so the emitted GNSS matches the real circuit.
- **No elevation:** the TUM data is 2D. Yas is relatively flat; elevation stays optional until surveyed.
- **Porsche 992 CAN:** no complete public DBC. Placeholders until the car can be sniffed.
- **RACELOGIC module database:** the thermal and ADC03 CAN definitions must be obtained (paper reference [63]). Infrared sensor output rate determines the real `bus.thermal` load.
- **Car parameters:** approximate until measured.
- **Licensing:** TUM racetrack-database is LGPL-3.0. Check obligations before any external distribution.
- **Windows timing:** real-time pacing depends on the high-resolution sleep; verify on the target laptop.
- **ZeroMQ `ipc://` on Windows:** not relied on; loopback TCP is the Windows default.
- **Real-car time synchronisation:** GPS or PTP discipline of the reaction computer is undecided.

## 15. References

- The paper, sections "Technical data and timing contract", "Combining VBOX with the TRKEYE brains", "Three lap calibration programme", "Predetermined braking baseline procedure".
- TUM racetrack-database: https://github.com/TUMFTM/racetrack-database
- RACELOGIC documentation cited in the paper: [3] Sigma CAN Output, [8] VBO structure, [19] VBO files, [53] ADC03 configuration, [57] VBOX 4 CAN output, [63] module CAN database.
