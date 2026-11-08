# Prompt: TRKEYE system architecture design

You are a senior systems architect with experience in real-time vehicle telemetry, motorsport data acquisition and AR display pipelines. Produce the overall software architecture for TRKEYE.

## Context

TRKEYE displays an adaptive racing line, braking marker and turn-in/exit cues inside a custom Eye-Lights helmet, so the line appears painted on the circuit. Vehicle state comes from a RACELOGIC installation in a Porsche 992 GT3 RS at Yas Marina:

- VBOX 4 Dynamics with dual antennas and IMU05-S: primary pose, 100 Hz RTK.
- VBOX Video HD2 with Sigma RTK (10 Hz): video-linked reference lap.
- CAN Gateway, ADC03 (hydraulic pressure, pedal force), 4 infrared tyre and 4 infrared rotor sensors, 4 suspension-travel sensors at 500 Hz.

Two "brains":

- **Heavy brain** in the paddock: reviews sessions, fits response models, generates trajectories.
- **Reaction brain** in the car: runs the live guidance loop and renders the line from the predicted eye viewpoint.

Project sequence: today only documents exist. Next comes the car with the full hardware (real logs follow), then the Eye-Lights kit. Milestone 1 is track-day readiness: the first session must produce decodable, time-aligned data. A configurable simulator that emits data structurally identical to the real sensors is being built in parallel.

## Read first

1. `docs/superpowers/specs/2026-11-08-trekeye-simulator-design.md` — the approved simulator spec. Its decisions (Python 3.12 monorepo with uv and pnpm workspaces, sensors as plugins inside one `sim` process, service separation at the physical-bus level, the four communication planes, `contracts/` as single source of truth) are settled. Do not reopen them unless you find a concrete defect. If you do, state the defect, the evidence and the proposed change separately.
2. The technical paper *RACELOGIC VBOX Integration with TRKEYE* (5 November 2026), if available.

## Task

Design the architecture of the whole TRKEYE application, not only the simulator. Cover today's services (`sim`, `logger-emu`, `ingest`, `gateway`, `dashboard`) and the future ones (heavy brain, reaction brain, helmet tracking, renderer and Eye-Lights integration, session and reference library, car-to-paddock link). Show exactly how the simulator is replaced by real hardware without changing consumers.

## Deliverables

Write one Markdown document, `docs/architecture/overview.md`, with Mermaid diagrams:

1. **C4 context diagram:** driver, test engineer, RACELOGIC hardware, Eye-Lights helmet, TRKEYE system, external tools (Circuit Tools, VBOX Setup).
2. **C4 container diagram:** every service and data store, with the topics and APIs between them.
3. **C4 component diagrams** for `sim`, `ingest` and `gateway`.
4. **Deployment views:**
   - Development laptop (everything on one Windows machine).
   - Track day (in-car reaction computer, pit laptop, paddock server).
   - Future full system with the helmet.
5. **Service catalogue** (table): responsibility, inputs, outputs, state held, process/threading model, language, and what changes between simulator and real car.
6. **Data lifecycle:** sensor → bus → logger file → ingest → aligned session → session model → trajectory package → reaction brain → rendered cue. Include versioning and provenance at each step, as the paper requires.
7. **Monorepo rules:** package list, allowed dependency directions (which package may import which), contracts code generation, build, test and CI layout, versioning and release.
8. **Cross-cutting concerns:** time and clocks, configuration, observability (logs, metrics, health), degradation and error handling (what happens when RTK, a bus or the gateway fails), minimal security, testing strategy per layer.
9. **Architecture decision records:** one short entry per significant decision, with options considered and the reason for the choice. Include the decisions already in the spec.
10. **Risks and open questions**, each with an owner role and the evidence needed to close it.

## Constraints

- Development and track-side laptop run Windows. The in-car reaction computer may run Linux; state where this matters.
- Python 3.12 for services, TypeScript for the dashboard. Do not assume the reaction brain stays in Python; keep its boundary language-neutral.
- Prefer the smallest design that satisfies milestone 1 and leaves clear extension points. Mark anything beyond milestone 1 as "later".

## Acceptance criteria

- Every service and topic in the spec appears in the container diagram, with a producer and at least one consumer. No orphan topics.
- The simulator-to-real-hardware swap is shown explicitly for every bus and logger file.
- Every future service has a defined interface boundary, even if its internals are "later".
- Each diagram fits on one screen and is readable without the text.
- Claims about RACELOGIC or Eye-Lights behaviour cite the paper or the vendor documentation. Estimates are labelled as estimates.
