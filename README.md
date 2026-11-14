# LEAR-Mesh

**Link-quality and Energy-Aware Routing Mesh** is an embedded-systems project
developing gateway-free text communication over a custom LoRa mesh, with BLE
links between phones and handheld nodes. It uses the LoRa physical layer,
not LoRaWAN.

The **proposed** prototype comprises five fixed relays and two handheld nodes.
The intended routing metric combines link delivery quality (ETX), battery state,
and queue occupancy. Routing, reliable forwarding, self-healing, BLE messaging,
and solar-aware power management are future milestones, not working features.

## Current status

M0 provides the initial development scaffold:

- Portable C11 hardware abstraction interfaces and a deterministic fake HAL.
- Bounded, structured diagnostics with explicit error handling.
- Host tests using CMake, Ninja, and CTest.
- ESP32-S3 adapter and minimal relay/handheld application scaffolds.
- Kconfig settings for node role, node ID, and diagnostic level.

Host tests passed in the Debian 13 cloud environment, including a separate
AddressSanitizer/UndefinedBehaviorSanitizer run. **M0 acceptance is incomplete:**
neither application has been cross-build validated. SDK installation was blocked
by HTTP 403 while downloading Espressif's Python constraints from
`dl.espressif.com`.

No hardware is assembled. Radio, BLE, routing, power consumption, and on-device
behavior are untested. There are no measured LEAR-Mesh performance results.
See [STATUS.md](STATUS.md) for the detailed evidence and outstanding work.

## Hardware and SDK

| Item | Current decision |
| --- | --- |
| MCU module | ESP32-S3-WROOM-1-N8R8 on a custom board |
| Radio module | Wio SX1262; exact variant and wiring pending |
| Board schematic and pin map | Undecided |
| ESP-IDF | v6.1, commit `fff9895c82d744c7237be8847347bdd1b07c6643` |
| Developer's local OS | Ubuntu 26.04.1 LTS; local compatibility unverified |
| M0 configuration | 8 MB flash, PSRAM disabled, no radio or BLE initialization |

No separate NimBLE or radio-library version has been selected for M0.

## Run host tests

Install a C compiler, CMake 3.22 or newer, and Ninja, then run from the repository
root:

```sh
./tools/test-host.sh
```

The script configures a Debug build with warnings treated as errors, builds the
host test executable, and runs CTest. Tests cover HAL behavior, diagnostic fields,
invalid inputs, identifier boundaries, integer limits, and output failures.
ESP-IDF and physical hardware are not required for these tests.

## Build the ESP32-S3 applications

Install the pinned ESP-IDF release and its ESP32-S3 tools outside this repository,
following the [M0 setup instructions](docs/M0.md#reproduce). Keep TLS, checksum,
and dependency-constraint verification enabled.

After sourcing that SDK's `export.sh`, run:

```sh
./tools/build-targets.sh
```

The script checks the SDK commit and builds `apps/relay` and `apps/handheld` for
`esp32s3`. These target commands remain pending validation because SDK installation
is incomplete. A successful cross-build will not establish hardware functionality.

## Repository layout

```text
apps/relay/               Minimal relay ESP-IDF project
apps/handheld/            Minimal handheld ESP-IDF project
components/lm_hal/       Portable HAL contracts
components/lm_hal_esp32s3/ ESP-IDF HAL adapter
components/lm_diag/      Structured diagnostics
components/lm_config/    Kconfig options
test/host/               Host tests and fake HAL
tools/                   Host-test and target-build scripts
docs/M0.md               Design decisions, setup, and pending bench procedure
```

## Development milestones

The **proposed** sequence is M0 scaffold/HAL/diagnostics, M1 radio characterization,
M2 packet codec, M3 MAC and hop reliability, M4 neighbor management, M5 routing
and forwarding, M6 healing, M7 messaging, M8 power management, M9 BLE integration,
and M10 experiments. Work proceeds one milestone at a time with recorded evidence.

- [AGENTS.md](AGENTS.md): architecture constraints and development workflow.
- [CLAUDE.md](CLAUDE.md): proposed design and evaluation baseline.
- [STATUS.md](STATUS.md): implemented work, executed checks, and next actions.
- [docs/M0.md](docs/M0.md): M0 scope, contracts, and acceptance criteria.

## Before radio operation

The developer reports India **865–868 MHz** as the verified operating band.
Detailed EIRP, antenna, duty-cycle/LBT conditions, and authoritative references
remain to be recorded before transmission. Finalize the schematic, module variant,
and programming interface before board bring-up. M0 performs no RF transmission.

The proposed baseline has no mesh authentication or encryption; confidentiality
and production security are not implemented.
