# mesh

This repository is the base workspace for Meshtastic-related projects and local convenience scripts.

It uses `uv` with `pyproject.toml` to manage the Python environment and dependencies, including the `meshtastic` CLI.

The `meshtastic-web/` directory is a local clone of the upstream Meshtastic web client used for local development and testing. It is intentionally ignored by Git so this repository stays focused on code written here rather than vendoring the upstream web app.

Planning notes for the weather telemetry setup are in `docs/weather-telemetry-plan.md`.
