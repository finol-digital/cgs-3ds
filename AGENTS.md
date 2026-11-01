# AGENTS.md

Guidelines for agents working on cgs-3ds.

- Read [docs/PLAN.md](docs/PLAN.md) first. It records the decisions (D1–D6) and phases.
- Related repos: `Card-Game-Simulator` (Unity app; reference for formats and table
  behavior), `FinolDigital.Cgs.Json` (shared formats; the converter and protocol will move there),
  `Card-Game-Simulator-XR`.
- Do not copy Unity/C# app code into the 3DS client. Reproduce behavior from the CGS data
  formats and observed behavior.
- No Nintendo SDK (CTR-SDK) material, keys, or proprietary firmware may enter this repo.
- Test on an emulator first. Never script firmware-level changes on the debug unit; those are
  manual, documented, and reversible (NAND/OTP backed up first).
- All source files start with the MPL 2.0 header used by CGS:
  `This Source Code Form is subject to the terms of the Mozilla Public License, v. 2.0...`
