# feast-stack

FEAST is a modular machine control stack. It connects embedded BREAD hardware to a process
orchestration runtime through a strictly layered C/C++ architecture — separating transport,
protocol, hardware drivers, simulation, and orchestration into independent focused components.
The runtime exposes unified state and control over HTTP REST and optional InfluxDB telemetry,
enabling automation logic to be written independent of hardware details.

## Repos

| Repo                                                                         | Role                                                                                                                          |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| [CRUMBS](https://github.com/FEASTorg/CRUMBS)                                 | I2C framing transport (C, runs on Arduino MCUs and Linux)                                                                     |
| [bread-crumbs-contracts](https://github.com/FEASTorg/bread-crumbs-contracts) | BREAD device application layer: type IDs, opcodes, payload layouts, caps schema (header-only C)                               |
| [anolis-provider-bread](https://github.com/FEASTorg/anolis-provider-bread)   | ADPP hardware provider for BREAD devices over CRUMBS (Linux hardware builds; foundation builds are cross-platform)            |
| [anolis-provider-sim](https://github.com/FEASTorg/anolis-provider-sim)       | ADPP provider for simulated devices with physics modeling and fault injection (reference implementation and development tool) |
| [anolis](https://github.com/FEASTorg/anolis)                                 | Runtime: orchestrates providers, maintains state cache, routes control, exposes HTTP API, drives automation                   |
| [anolis-protocol](https://github.com/FEASTorg/anolis-protocol)               | ADPP protobuf schema — a standalone repo vendored as a git submodule into each consuming repo                                 |

## Docs

- [docs/stack.md](docs/stack.md) — layer contracts, dependency rules, design principles
- [docs/adpp.md](docs/adpp.md) — ADPP protocol reference (framing, RPCs, schema, status codes, provider contract)
- [docs/devices.md](docs/devices.md) — BREAD device catalog (RLHT and DCMT signal/function/capability reference)

## Dev Setup

For a new Linux machine:

```sh
sudo apt update && sudo apt install -y \
  build-essential cmake ninja-build pkg-config git curl zip unzip tar

git clone https://github.com/microsoft/vcpkg ~/vcpkg
~/vcpkg/bootstrap-vcpkg.sh
echo 'export VCPKG_ROOT=$HOME/vcpkg' >> ~/.bashrc
source ~/.bashrc
```

The expected local workspace layout:

```text
repos_feast/
├── anolis/
├── anolis-provider-bread/
├── anolis-provider-sim/
├── CRUMBS/
├── bread-crumbs-contracts/
└── linux-wire/              # required for provider-bread hardware builds
```

Each repo has its own `docs/build.md` with configure, build, and test commands.
