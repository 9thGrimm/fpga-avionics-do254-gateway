# Fault-Tolerant Multi-Clock Avionics Telemetry and Control Gateway

A Zybo Z7-20 (Zynq-7000) research project exploring independent source clocks, telemetry validation, safe source selection, fault handling, and PS/PL integration.

The development record follows a DO-254 / FAA AC 20-152A-informed, DAL B representative exercise. This bench-top project does not claim certification or a certified hardware DAL.

## Status

Background research and concept definition. No gateway RTL has been baselined.

## Project documents

- [Project plan](docs/project-plan.md)
- [Research notes](docs/research-notes.md)
- [Decision log](docs/decisions.md)

## Working layout

- `assurance/`: planning, reviews, traceability, configuration records
- `requirements/`: system and FPGA requirements
- `architecture/`: diagrams, interfaces, clock/reset and CDC inventories
- `rtl/`, `constraints/`, `scripts/`: design sources and reproducible builds
- `verification/`, `hardware_tests/`: tests, procedures and results
- `software/`: Zynq PS demonstration code
- `releases/`: tagged release manifests and evidence summaries

Generated Vivado files belong under `build/` and are excluded from version control. Record the board revision, Vivado version, IP configuration, and exact source baseline for each hardware result.
