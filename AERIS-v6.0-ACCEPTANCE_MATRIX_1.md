# AERIS v6.0 · Acceptance matrix

Written **before** implementation (9 Oct 2026) from `evidence/v6.0-inputs/AERIS-Claude-Implementation-Prompt.md`, then
filled with the evidence that was actually produced. A row without executed evidence would stay `planned`; a check that
cannot run in the build environment is `blocked` with its reason. Statuses never upgrade a capability that needs
physical hardware: the last column says what remains for a real cube.

Status words: `planned` · `implemented` (code exists) · `software-tested` (executed tests pass) ·
`hardware-tested` (on a real board / cube) · `blocked` (could not be executed here; reason given).
**No row is `hardware-tested`: nothing in v6.0 ran on a physical board, sensor, driver, motor or cube.**

Evidence names: suites of `scripts/run_tests.py` (the definitive run is `test-logs/`; `SUMMARY.md` has the counts),
`evidence/v6.0-verification/…` (probes, negative controls, builds), the fresh-directory release validation
(`evidence/v6.0-release/`, `scripts/validate_release.py`). "v6.0 suites" means: `drift`, `backend` (pytest),
`firmware:host`, `firmware:qemu`, the JavaScript suites, `live`, the browser suites.

## 1. The four v5.2 review findings (reproduced on unmodified v5.2 first)

| ID | Requirement | Evidence (executed) | Status | Remaining for a real cube |
|---|---|---|---|---|
| V52-1a | Reproduce the reconnect gap | `evidence/v6.0-verification/reproduction-v5.2/`: the reviewer's `websocket_gap_probe.cjs` on unmodified v5.2 delivers `[2]`, never requests frame 1, no gap event | software-tested | — |
| V52-1b | Race-safe resume / catch-up | `js:resume` R1–R14 (same-process reconnect, process restart, frames during handshake/catch-up, duplicates, old-stream answers, several history pages, buffer overflow, unretained history, unreadable history, slow subscriber, live edge, polling, close during catch-up, the reviewer's fixture with history) | software-tested | — |
| V52-1c | Explicit unrecoverable history | `js:resume` R8, R9, R12 (gap with stream, range, count; cursor moves only then); after-v6 run of the reviewer's probe: frame 1 requested, reported as a gap because the fixture's history is empty (`after-v6/`) | software-tested | — |
| V52-1d | Real backend, deterministic | `live` L15 (gated socket against a real backend process); negative control: L15 **fails with v5.2's adapter, passes with v6's** (`after-v6/negative-control-L15/`); the reviewer's fixture with a history that has frame 1: v6 `[1, 2]`, v5.2 `[2]` | software-tested | — |
| V52-1e | L14 unchanged | the L14 block of `live_backend.test.js` is unchanged since v5.2 (only the shared helpers and L15 changed); `live` passes | software-tested | — |
| V52-2a | Reproduce live provenance promotion | `reproduction-v5.2/`: the reviewer's `provenance_probe.py`, all four artificial lineage cases `live: true`, `synthetic: false` | software-tested | — |
| V52-2b | One conservative classifier | `backend` `test_live_provenance.py` (status, `/capabilities`, WebSocket batches, recordings, comparison eligibility share one rule; `unknown` distinct from synthetic; `physical_verified` false); the reviewer's probe on v6: synthetic ancestry never live, unknown trust kept (`after-v6/`) | software-tested | — |
| V52-3a | Reproduce stop-timer kill | `reproduction-v5.2/`: the reviewer's `stop_timer_probe.cjs`, the old timer kills the replacement | software-tested | — |
| V52-3b | Scoped, idempotent stop | `js:procs` P1–P7 (including real processes); `after-v6/stop_timer_probe_v6.cjs` (the reviewer's scenario on the shared helper: replacement never killed); `live` uses the helper | software-tested | — |
| V52-4a | Reproduce audit miss | `reproduction-v5.2/`: the reviewer's `export_probe.py current` misses COPY_2 | software-tested | — |
| V52-4b | Transitive read-only audit | `backend` `test_export_audit.py` (10 cases: the reviewer's chain, three legacy generations, v5.2 descendants, missing intermediates and originals, cycles, ambiguity, v5 replays, the command line changes nothing — database byte-identical); the reviewer's probe on v6: COPY_2 confirmed (`after-v6/`); `site/docs/MIGRATION_V6.md` §6 | software-tested | — |
| G1–G4 | Earlier fixes preserved | the suites of v5.2 run unchanged in the definitive run (`test_export_units.py`, `test_lineage_bounded.py`, the runner's skip accounting in `test_run_tests.py` — only new tests added, `browser_v52`) | software-tested | — |

## 2. Virtual-first website and the build journey

| ID | Requirement | Evidence (executed) | Status | Remaining for a real cube |
|---|---|---|---|---|
| VF-1 | Virtual cube stays the default | `browser_v6` (Home's primary action; the virtual pages load nothing of the device stack; no script errors), the v2–v5.2 browser suites from `file://` | software-tested | — |
| VF-2 | Research flow with a real backend | `live`, `browser_v4`, `browser_v5`, `browser_v51`, `browser_v52`; release validation steps experiment … telemetry-generations | software-tested | — |
| VF-3 | Entry points | `browser_v6` (Try the Virtual Cube primary; Build and Connect in navigation, menu, footer; a route back to the virtual cube on both pages) | software-tested | — |
| BLD-1 | Versioned kit content | `drift` (kit documents and BOM copy generated from `hardware/bom/`), `backend` (kit schema), `browser_v6` (readiness ladder at *candidate*, every BOM item with its sources) | software-tested | the kit itself is a *candidate*: motors, rotors, frame, harness, power system not selected or designed (`hardware/COMMISSIONING.md` §1) |
| BLD-2 | Truthful assets | `browser_v6` (downloads exist with SHA-256; unavailable ones show the reason and no link or button); every step's illustration is `none` with the note "No photograph or drawing of an AERIS A1 build exists: none is shown" (kit document, generated and drift-checked); the site's renders and video are labelled AI-generated (`browser_check`) | software-tested | CAD, drawings, PCB, harness drawing, photographs of a build do not exist |
| BLD-3 | Steps | kit document schema (`backend`), `browser_v6` (blocked steps say why and cannot be ticked; step actions open the matching Connect step) | software-tested | the steps for the unselected parts are blocked |
| BLD-4 | Saved progress per kit revision | `js:connect` (build progress), `browser_v6` (kept per revision; an unknown earlier revision does not carry over and is explained; clearing) | software-tested | — |
| BLD-5 | Accessibility | `browser_v6` (one h1 per page; keyboard arrows, Home, End on the device steps; phones at **375 and 390 px** without horizontal scrolling; the menu shows every entry) | software-tested | not audited with assistive technology |
| BLD-6 | Synthetic setup demonstration | `browser_v6` (the whole journey with the WebAssembly synthetic device, labelled SYNTHETIC — NOT A CUBE everywhere) | software-tested | — |

## 3. Profiles, calibration, recipes

| ID | Requirement | Evidence (executed) | Status | Remaining for a real cube |
|---|---|---|---|---|
| PRF-1 | Schemas + APIs | `backend` `test_devices_api.py`, `test_devices_unit.py` (profiles, recipes, calibrations, kits, configurations, commands); the firmware manifest (`drift`: signature and parts) | software-tested | — |
| PRF-2 | Validation | `backend` `test_profile_validation_rules` (non-finite values, inertia symmetric / positive definite / **triangle inequality** for the body and components, unit wheel axes, coplanar axes, distinct nodes, signs, the IMU mounting a proper signed permutation, voltage and current limits consistent, qualification refused, value sources, gravity; exactly three wheels — other topologies refused with the reason) | software-tested | — |
| PRF-3 | Immutability | `backend` (a changed profile is a new revision, an unchanged one is not; configurations, recordings (`device_link.config_hash`), twin runs (`twin.profile` with its hash) and comparisons (`run_config_hash`, `recording_hash`) reference what they used) | software-tested | — |
| PRF-4 | No self-promotion | `backend` (a user profile cannot be `qualified` (422); after pairing, calibration and an experiment of a device, its profile's value sources are unchanged and the twin run is NOMINAL (`test_synthetic_device_end_to_end`); a comparison with a synthetic recording is never admissible), the release validation's device journey | software-tested | qualification needs reviewed hardware evidence |
| PRF-5 | One parameter source | `backend` `test_twin_runs_are_labelled_through_the_parameter_set_coverage_rule`, `test_twin_run_is_labelled_by_the_profiles_value_sources` (the profile's groups declared measured become the run's parameter set at the engine's paths; the coverage rule of PARAMETERS.md labels the run; `datasheet`/`estimated` are not measurements; never the backend's global set) | software-tested | measured values exist only after the measurements of `hardware/COMMISSIONING.md` §2 |

## 4. Firmware

| ID | Requirement | Evidence (executed) | Status | Remaining for a real cube |
|---|---|---|---|---|
| FW-1 | Compilable project | portable C11 core + host build (**make**, not CMake: `firmware/Makefile`, `-Wall -Wextra -Wpedantic -Werror`) — `firmware:host`; ESP-IDF v6.0.3 board project (CMake) built in three variants with no warning (`evidence/v6.0-verification/firmware-build/`) | software-tested | a board |
| FW-2 | State machine | `firmware:host` (states, guards, self-test to READY, local enable, normal stop, FAULT latch and clearing, update, commissioning) | software-tested | on the board |
| FW-3 | Sensing/estimation | `firmware:host` (LSM6DSOX register driver against its model; cf3d estimator parity with the backend: max |Δq| 4.1e-6 rad over 800 samples) | software-tested | a real IMU: noise, bias, mounting, an independent attitude reference |
| FW-4 | Controllers | `firmware:host` (edge LQR max |ΔT| 1.1e-6 N·m, corner MIMO LQR 1.6e-6 N·m over 60 cases each, gains from the backend's design) | software-tested | gains designed on a measured profile |
| FW-5 | Allocation and limits | `firmware:host` (torque limit = kt · current limit per wheel, no accelerating torque at the speed limit, braking allowed, non-finite requests zeroed, saturation flags; the experiment's result reports the saturated control steps per wheel); negative control removes the limits and the count (`firmware-fault-tests/`); the driver's current limit is configured over CAN | software-tested (**partial**: the motor's voltage/back-EMF limit is not modelled in the firmware's allocation — the driver applies it) | the driver's real limits; qualified current, speed and thermal limits |
| FW-6 | Faults / watchdog | `firmware:host`: local stop, tilt limit, overspeed, and — injected into the models — over-temperature, over-current, under- and over-voltage, lost driver, driver error, stale encoder estimates, lost IMU / invalid estimate, late control steps (LOOP_OVERRUN), a failed check in READY; each latches FAULT, removes driver power and fails the experiment; clearing needs the cause gone and a long press; negative control: 13 of 13 removed checks detected (`firmware-fault-tests/`). Not reachable through the models: a non-finite IMU value (guarded in `read_imu`) | software-tested | every fault on the real hardware in the fixture |
| FW-7 | Communication-loss policy | `firmware:host` (link-loss: continue up to `limits.link_loss_max_s`, then a normal stop) | software-tested | in the fixture |
| FW-8 | Closed-loop harness | `firmware:host` (edge and corner balance of the engine's plant with sensor noise, recovery, saturation, faults); `backend` end-to-end comparison of a synthetic device recording with Simulate My Cube | software-tested | a real cube |
| FW-9 | Drivers | `firmware:host` (LSM6DSOX and ODrive CANSimple drivers against register/bus models; another chip at the IMU address is never used; unsupported IMU and wheel drivers refused by the configuration; missing parts reported, nothing fabricated); `firmware:qemu` (IMU and drivers absent in the emulator, reported absent) | software-tested | the real sensor and drivers |
| FW-10 | Actuation default | `firmware:host` (release build never actuates; commissioning: dead-man, caps, one wheel); the release manifest refuses builds with actuation; the installer refuses manifests with actuation (`js:connect`, `browser_v6`); the experimental image is not distributed | software-tested | commissioning in the fixture with the hardware stop chain |
| FW-11 | Timing | host timing of the control step (`evidence/v6.0-verification/firmware-timing/`: balancing: mean ≈ 2.1 µs, p99 ≈ 13 µs, one maximum of 0.55 ms in 4000 steps on a shared 2-CPU host); the QEMU run reports emulator timing only | software-tested (host) / **blocked** (board) | the ESP32-S3's step time, jitter and overruns at 500 Hz with Wi-Fi active |
| FW-12 | Board image | ESP-IDF v6.0.3 builds (logs, sizes, SHA-256), signed release manifest with the firmware source hash (`drift`), `firmware:qemu` (22 checks on the emulated ESP32-S3) | software-tested (emulator) | flashing and running on a physical ESP32-S3 |

## 5. Connections, devices, owners, commands

| ID | Requirement | Evidence (executed) | Status | Remaining for a real cube |
|---|---|---|---|---|
| CON-1 | Web Serial | `browser_v6` with a **scripted Web Serial port** (a labelled test double: port choice filtered to Espressif's USB, the device link over it, disconnect closes the port); `js:connect` (framing, negotiation, link session drop and re-open) | software-tested (test double) | a real USB serial port and board |
| CON-2 | Companion | `backend` `test_companion_over_a_real_serial_device` (pyserial on a **real pseudo-terminal** of the native synthetic device: pairing after the local press, lease, commands), the end-to-end tests (TCP), release validation device journey | software-tested | a USB-serial driver and cable |
| CON-3 | Outbound WSS gateway | `backend` end-to-end (device-initiated, key-authenticated, observer while USB is attached, telemetry continued across route changes and reconnections without missing frames, revocation refused 4401), `test_devices_telemetry.py` (ring gaps, session ends), `test_gateway_and_companion_over_tls` (**TLS on loopback** with a test CA; a client without it is refused by certificate verification) | software-tested | the board's own TLS client (ESP-IDF esp-tls + CA bundle), a public certificate, real networks |
| MD-1 | Multi-device | `backend` (per-device sessions and hubs; two concurrent devices stay connected; the global v1 telemetry bridge is off in jwt mode) | software-tested | — |
| MD-2 | Isolation | `backend` `test_two_owners_two_concurrent_synthetic_devices`, `test_jwt_requires_tokens_and_isolates_owners`, `test_jwt_websocket_tickets` (REST, WebSocket, commands, pairing, recordings, datasets, jobs, runs across owners → 404) | software-tested | — |
| PAIR-1 | Pairing | `backend` (single-use proofs, forged signatures and missing local confirmation refused, revocation), `firmware:host` (local proof over USB, network authentication, observer role, timeout, unpair with local confirmation), `firmware:qemu` (key generated on the device) | software-tested | on a board |
| CMD-1 | Command contract | `firmware:host`, `backend` (ids, sessions, lease epochs, ttl from receipt, statuses, history, delivery ≠ execution) | software-tested | — |
| CMD-2 | Safety of commands | `firmware:host` (stale, replayed, duplicate, out-of-state, unsupported refused; a stale request never starts; stop idempotent; lease handover), `backend` (restart marks open commands expired, never resent; takeover epochs; USB-local actions refused remotely) | software-tested | — |
| CMD-3 | Local presence | `firmware:host`, `backend` e2e, `browser_v6`, release validation (an experiment waits for ARM on the cube; bounded duration; link loss ends it) | software-tested | the physical ARM and stop controls |
| PUB-1 | Public mode | `backend` (`test_device_api_disabled_on_a_public_bind_without_jwt`, `test_auth_config_refuses_unsafe_settings`, the development fixture refused off loopback; v1 owner scoping in jwt mode) | software-tested | a real identity provider and deployment (none performed) |

## 6. Installer, twin, persistence

| ID | Requirement | Evidence (executed) | Status | Remaining for a real cube |
|---|---|---|---|---|
| INS-1 | Installer logic | `js:connect` (manifest canonical form, trust levels, tampering, image hashes, chip/flash checks, interruption, identity afterwards, never while BALANCING) and `browser_v6` (development-key consent, scripted flasher, a modified image refused before writing, the board's hello checked) — **scripted flasher, a labelled test double** | software-tested (test double) | flashing a real board |
| INS-2 | No placeholder images | `drift` (the signed release verifies; its source hash equals the sources; the kit lists only files that exist with their SHA-256); only the release image is published | software-tested | — |
| TWN-1 | Simulate My Cube | `backend` (profile revision + recipe (+ calibration) → engine configuration; refusals; the run's label from the profile's value sources) | software-tested | a measured profile |
| TWN-2 | Record → simulate → compare → export/reimport | `backend` e2e and release validation device journey (synthetic recording, twin run, comparison never admissible, export/reimport keeps the content hash, NDJSON hello synthetic); `browser_v6` (local and backend flows) | software-tested | a declared-physical recording under a predeclared protocol, reviewed by a person |
| DB-1 | Migration / backup / restore | `backend` (`test_persistence.py`, `test_recovery.py`: schema 2 → 3 additive, refusal of a newer schema; `test_backup_and_restore_keep_devices_commands_and_documents`); release validation backup step | software-tested | — |
| NUM-1 | Numerical reproducibility | `studies/verify_campaign.py`: 32 rows re-run — 32 identical, 0 outcome changes, unchanged tolerances (`evidence/v6.0-verification/numerical/`) | software-tested | — |

## 7. Release

| ID | Requirement | Evidence (executed) | Status | Remaining |
|---|---|---|---|---|
| REL-1 | Runner accounting | `backend` `test_run_tests.py` (new suites `drift`, `firmware:host`, `firmware:qemu` counted exactly; missing tools → NOT RUN with the reason; a not-run check makes the run INCOMPLETE) | software-tested | — |
| REL-2 | Full run | `test-logs/`: every suite on the final product commit `91f78f3` — 26 suites, **1105 passed, 0 failed, 0 skipped, 0 not run**, PASSED (full validation on Linux); the first full run at `56dd89e` is kept (`evidence/v6.0-verification/full-run-1-56dd89e/`) | software-tested | — |
| REL-3 | Clean-install validation of the exact archive | `scripts/validate_release.py`: the candidate archive (`b5a2967`) **21 of 21 steps** in a fresh directory, including the v6.0 device journey, drift, firmware host tests and browser_v6 (`evidence/v6.0-release/`); the delivered archive's own validation is delivered next to it | software-tested | — |
| REL-4 | Integrity | `scripts/compare_archive_bundle.py`: the candidate archive equals its commit (1 179 files, same bytes) against a verified bundle of the complete history (`evidence/v6.0-release/`); the delivered archive is compared with the tag `aeris-v6.0` and delivered with its check and SHA-256 | software-tested | — |
| DOC-1 | Documentation | `site/docs/MIGRATION_V6.md`, `CONNECT.md` (local launch, public deployment), `DEVICES.md` (API, documents), `DEVICE_LINK.md`, `FIRMWARE_INSTALL.md`, `firmware/README.md`, `hardware/` (BOM, build guide, integration, sizing, commissioning list), readiness tables in `SOFTWARE_COMPLETION_REPORT.md` | implemented | — |
