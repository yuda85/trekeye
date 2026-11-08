# TRKEYE

Adaptive racing-line guidance for the Eye-Lights helmet, driven by RACELOGIC VBOX vehicle telemetry.

This monorepo holds the TRKEYE application services: the configurable sensor simulator, data ingest and time alignment, the gateway and the web dashboard.

## Layout

```
packages/      Python services and shared libraries (uv workspace)
apps/          Front-end applications (pnpm workspace)
contracts/     Single source of truth for wire formats, DBC files and schemas
tracks/        Track packages
cars/          Vehicle parameter files
scenarios/     Simulator scenarios
docs/          Specs, design prompts and decisions
```

## Design

- Simulator design spec: [docs/superpowers/specs/2026-11-08-trekeye-simulator-design.md](docs/superpowers/specs/2026-11-08-trekeye-simulator-design.md)
