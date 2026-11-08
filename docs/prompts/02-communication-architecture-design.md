# Prompt: TRKEYE communication architecture design

You are a senior engineer specialising in low-latency data distribution, real-time systems and telemetry pipelines. Design the communication architecture for TRKEYE so that every consumer receives data at the required update rate and latency, with measured evidence.

## Context

TRKEYE turns RACELOGIC vehicle telemetry into a racing line rendered inside an Eye-Lights helmet. A configurable simulator (Python, one `sim` process with sensors as plugins) emits data structurally identical to the real installation: per-bus CAN frames with receipt timestamps from the receiver's clock, logger files, and a separate ground-truth stream. The same consumers must later work unchanged on the real car.

Development and track-side machines run Windows. Services are Python 3.12; the dashboard is TypeScript with three.js in the browser. A future in-car reaction brain may be written in C++ or Rust.

## Read first

1. `docs/superpowers/specs/2026-11-08-trekeye-simulator-design.md`, especially §4 (architecture) and §5 (communication architecture). Treat its rate budget, four planes, batched columnar wire format and benchmark gate as the starting point. Refine them with evidence. Do not discard them without stating the defect and the measurement that shows it.
2. `docs/architecture/overview.md` if it exists.
3. RACELOGIC documentation for the VBOX 4 CAN output, Sigma CAN output, ADC03 configuration and the module CAN database (infrared sensors).

## Required rates and budgets (from the spec)

| Topic | Real-time rate (estimate) | Transport latency budget | Semantics |
|---|---|---|---|
| `bus.vbox4` | ~1,600 frames/s (100 Hz × ~16 IDs) | ≤ 1 ms p99 | Lossless, ordered |
| `bus.vehicle` | ~1,000 frames/s | ≤ 1 ms p99 | Lossless, ordered |
| `bus.thermal` | up to ~3,200 frames/s, 1 Mbit/s CAN | ≤ 5 ms p99 | Lossless, ordered |
| `bus.module` | ~400 frames/s | ≤ 1 ms p99 | Lossless, ordered |
| `log.vbox4` | ~2,100 samples/s (incl. 4 × 500 Hz analogue) | recording | Lossless |
| `sim.truth` | 1,000 records/s, ~1 KB each | ≤ 10 ms | Lossless to recorders, latest-value to UI |
| UI pose | 60 Hz | ≤ 50 ms to screen | Latest value |
| UI traces | 30–60 Hz decimated, spikes preserved | ≤ 100 ms | Droppable with gap markers |
| Control | events | ≤ 50 ms round trip | Reliable request/reply |

Batch mode must run the full pipeline at ≥ 20× real time, which is roughly 130,000–160,000 frames per second.

Physical context for the future helmet path (from the paper): at 200 km/h the car travels about 0.56 m in 10 ms and about 2.78 m in 50 ms. A 0.1° pointing error at 50 m is about 8.7 cm of lateral displacement.

## Tasks

1. **Verify the rate budget.** Replace estimates with documented values where RACELOGIC publishes them: message lists, rates, infrared sensor output rate, ADC03 polling. Recompute totals and CAN bus loads. Label anything still estimated.
2. **End-to-end latency budgets.** Allocate a budget per stage for each path:
   - sim → reaction-brain input queue (target ≤ 1 ms p99 for transport);
   - sim → gateway → browser → pixels;
   - future real path: measurement → bus → reaction brain → guidance → renderer → photons in the helmet, tied to the registration error figures above.
3. **Transport per plane.** For the in-process, live data, control and UI planes specify:
   - transport and socket patterns;
   - topic catalogue with reliability, history depth, conflation, priority and flush policy;
   - wire format (header and columnar body), sequence numbers and gap detection;
   - backpressure, slow-joiner handling, reconnect and restart behaviour;
   - failure modes and how each one shows up in health monitoring.
4. **Process and threading model.** For each Python process: threads, what runs on each, where the GIL is released, zero-copy paths, how real-time pacing avoids asyncio timer resolution on Windows, and the CPU budget per process at 1× and 2× nominal load.
5. **Browser protocol.** Binary framing, schema handshake, typed-array layout, M4 decimation, interpolation buffer delay, render loop decoupled from the network, per-client backpressure. Compare WebSocket with WebTransport and state when to switch.
6. **Large payloads.** Shared-memory ring buffer design for future camera frames (1080p at 30 fps), notification channel, ownership and slot reuse.
7. **Benchmark plan for `tools/bench/`.** Methodology (warm-up, `time.perf_counter_ns`, HDR latency histograms, Windows power plan, process priority, CPU affinity), parameter sweep, comparison of ZeroMQ, Zenoh and shared memory, pass criteria, and the report format to commit with hardware details.
8. **Future links.** Real CAN adapters (PCAN, Kvaser, SocketCAN) publishing the same batches; the language-neutral boundary to a C++ or Rust reaction brain; car-to-paddock link over intermittent wireless (store-and-forward, prioritised live summaries; Zenoh vs NATS JetStream); clock discipline of the reaction computer (GPS time from VBOX or PTP).

## Deliverables

1. `docs/architecture/communication.md` containing:
   - the verified rate budget and per-path latency budgets;
   - a data-flow diagram annotated with rates and sizes (Mermaid);
   - sequence diagrams for the real-time publish path, a control command and a UI stream (Mermaid);
   - the per-plane design, topic catalogue and failure-mode table;
   - the threading model and CPU budget;
   - the benchmark plan with pass criteria.
2. Draft schema files for `contracts/schemas/`: batch dtypes, truth record, topic catalogue with QoS, and UI stream layouts, in the YAML format the spec describes.

## Acceptance criteria

- Every topic has a rate, a message size, a latency budget, delivery semantics, a transport and a measurement method.
- Every budget adds up end to end, with stated margins.
- Every number traces to the paper, vendor documentation or a benchmark; estimates are labelled.
- The design meets the batch throughput target without per-frame messaging.
- Switching from simulator to real hardware changes only the producer of each topic.
