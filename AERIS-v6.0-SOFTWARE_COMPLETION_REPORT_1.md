# AERIS v6.0 · Software completion report

Scope: the upgrade specified in `evidence/v6.0-inputs/AERIS-Claude-Implementation-Prompt.md` (SHA-256
`a3fd9158…3cb9d`): the ready-made virtual cube stays the default; an optional **Build Your Own Cube** journey and an
optional **Connect My Cube** journey are added, with device profiles, guided calibration, firmware installation,
buildable firmware, connections, ownership, bounded commands, recording, simulation of the user's cube and comparison;
the four findings of the independent review of v5.2 (V52-1 … V52-4) are corrected. Baseline: tag `aeris-v5.2` =
`c28500044bab1ea73e82dff81f46fea8fccbf9f4`, archive `aeris-site-v5.2.zip` SHA-256 `5301331f…2931e370` and bundle
`7a86470c…6834f444` — verified against the handoff (781 files equal), not modified; v6.0 was built in a separate
checkout on branch `v6.0`. Preserved: the frontend design, `LocalSimulatorAdapter`'s contract, the research engine
(`aeris-research-engine 1.1.0`) and its numerical behaviour, records, schemas, `aeris-telemetry-ndjson/1`, `/api/v1` in
local mode, historical studies, user data, the v5.x fixes. Backend **1.2.0**, database schema **3** (additive),
firmware **aeris-fw 0.1.0**. Changes visible to clients: [MIGRATION_V6.md](site/docs/MIGRATION_V6.md).

Every statement below points to code and to evidence **executed by us** on the machine of §2, unless it is attributed
to the review. **No physical cube, board, sensor, driver or motor was built, connected, flashed, measured or balanced.**
Everything device-side was exercised with synthetic devices (the firmware core driving models of the IMU, the wheel
drivers and the engine's plant — natively, as WebAssembly in the page, or as the ESP32-S3 image in Espressif's QEMU)
and with labelled test doubles (a scripted Web Serial port and a scripted flasher). The acceptance criteria and the
evidence for each: [ACCEPTANCE_MATRIX_V6.md](ACCEPTANCE_MATRIX_V6.md).

## 1. Results

### 1.1 The review findings

| Finding | Status | Code | Tests | Before (v5.2) → after (v6.0), same probe |
|---|---|---|---|---|
| **V52-1** telemetry reconnect gap | fixed | `site/js/adapters/hardware-telemetry.js` (catch-up from the retained history, held live frames, explicit gaps) | `site/tests/resume.test.js` (R1–R14), `live_backend.test.js` L15 | reviewer's probe: `[2]`, frame 1 never requested → requested from the new stream (and reported as a gap because the probe's stub history is empty); with a history that has frame 1: `[1, 2]`; **L15 fails with v5.2's adapter, passes with v6's** |
| **V52-2** synthetic ancestry shown live-physical | fixed | `backend/aeris_backend/telemetry/lineage.py`, `stream.py`, `api/app.py` (one conservative classifier; `unknown` trust) | `backend/tests/test_live_provenance.py` | reviewer's probe: four lineage cases `live: true` → synthetic ones `live: false`, `trust: synthetic`; unknown `trust: unknown`, never live |
| **V52-3** stop timer kills the replacement backend | fixed (test infrastructure) | `site/tests/lib/procs.js`, `live_backend.test.js` | `site/tests/procs.test.js` (P1–P7) | reviewer's probe: replacement SIGKILLed → the same scenario on the shared helper: never |
| **V52-4** export audit one generation deep | fixed | `backend/aeris_backend/tools/export_audit.py`, `dbtool export-audit` (`aeris-export-audit/2`) | `backend/tests/test_export_audit.py` (10) | reviewer's probe: COPY_2 missed → confirmed with its chain; the database byte-identical before and after |

Before/after outputs: `evidence/v6.0-verification/reproduction-v5.2/` (unmodified v5.2) and
`evidence/v6.0-verification/after-v6/` (this release, with the negative control of L15).

### 1.2 Capability and readiness

No single completion percentage is given; each capability is classified by what was executed.

| Capability | Implemented | Software-tested | Hardware-tested | Blocked / open |
|---|---|---|---|---|
| Virtual cube as the default (Lab, controllers, disturbances, sweeps, analysis, `file://`) | yes | yes — the v2–v5.2 browser suites, `browser_v6` | — (simulation) | — |
| Build Your Own Cube page (kit A1-candidate, BOM with sources and check dates, sizing, steps, notes per revision) | yes | yes — `browser_v6`, `js:connect`, `drift` | — | the kit's motors, rotors, frame, harness, power system, controls and fixture are not selected or designed |
| Connect My Cube with the synthetic device in the page (WebAssembly) | yes | yes — `browser_v6`, `js:connect` | — (synthetic by definition) | — |
| Web Serial transport and firmware installer | yes | with a scripted port and flasher (labelled test doubles) | **no** | a board on USB |
| Device API `/api/v6`: owners, pairing, leases, bounded commands, calibration, configurations, telemetry, recordings, twin | yes | yes — `backend` (unit, API, end-to-end with the native synthetic device, two owners, TLS loopback, pty) and the release validation's device journey | **no** | — |
| Companion, browser relay, device gateway | yes | yes — synthetic device over TCP, a real pseudo-terminal and TLS on loopback | **no** | the board's TLS client, real networks |
| Authentication `local` / `jwt` with an external identity provider | yes | yes — with the development identity fixture (refused off loopback) | — | an identity provider and a public deployment (none performed) |
| Simulate My Cube and comparison | yes | yes — never admissible as validation evidence for synthetic data | — | a measured profile; declared-physical recordings under a predeclared protocol |
| Firmware core (states, estimator, controllers, allocation, protection, link, commands, telemetry, calibration, commissioning) | yes | yes — 38 host tests with a negative control of the protection tests | **no** | — |
| ESP32-S3 board layer and images (ESP-IDF v6.0.3) | yes | built (three variants); the test image passes 22 checks in QEMU | **no** | a physical ESP32-S3; board timing |
| Drivers: LSM6DSOX (I2C), ODrive CANSimple (TWAI) | yes | against register and bus models | **no** | the real sensor and drivers |
| Release signing and trust list | yes (development test key; production list empty) | yes — signature, trust, parts, source hash (`drift`) | — | a production key and release procedure |
| Hardware build content (BOM, sizing, integration, commissioning list) | yes (documents and tools) | content checks | **no** | everything physical: `hardware/COMMISSIONING.md` |
| Motor actuation | only in experimental builds made from source; off in the distributed image | yes — on the models (commissioning caps, local enable, protections) | **no** | qualified limits, a fall-catching fixture, a hardware stop chain, commissioning |

## 2. Environment and what was executed

Linux x86-64 (Ubuntu 24.04, kernel 6.18.44), Python 3.12.3 (virtual environment from the lock file: NumPy 2.5.3, SciPy
1.18.1, FastAPI 0.141.1, cryptography 50.0.2, PyJWT 2.15.1, pytest 9.1.1), Node.js 22.22.2, Playwright 1.56.0 with the
preinstalled Chromium, gcc 13.3.0 and OpenSSL for the host firmware, clang 18.1.3 for WebAssembly, ESP-IDF v6.0.3 with
xtensa-esp-elf-gcc 15.2.0 (esp-15.2.0_20251204) for the ESP32-S3, Espressif QEMU 9.2.2 (esp_develop_9.2.2_20260417),
esptool 5.5.0. Every service ran on 127.0.0.1 with temporary databases; no deployment, no account, no purchase, no
hardware, no user data. The firmware signing key lived outside the repository. Not executed by us: macOS, Windows,
non-Chromium browsers, a hosted CI service, a physical board, long soak tests, the complete campaign.

The test runner itself ran under the system's Python 3.11.15 (standard library only); every suite that needs the
backend used the locked 3.12.3 environment. The machine and tool versions as recorded at the definitive run, including
every installed Python package: `evidence/v6.0-verification/environment.json`. The container's host changed during the
work (the CPU model reads 2.80 GHz in early logs and 2.10 GHz later); no result depends on it except timing figures,
which are labelled as host measurements.

## 3. The review findings in detail

**V52-1.** The adapter now tracks, per stream, the last frame it delivered. After every (re)connection or stream
change it reads the backend's retained history for the missing range (page by page), holds frames arriving on the
socket until the history is in, and delivers each frame once and in order; when the history no longer has a range, a
`gap` event names the stream, range and count before the cursor moves. The reviewer's deterministic probe cannot show
recovery on v6 because its stub answers every history request with an empty list: v6 requests the new stream's
history from n = 0 and reports frame 1 as a gap. A marked copy of the probe whose stub returns the frames the fixture
describes delivers `[1, 2]` on v6 and `[2]` on v5.2. The live test L15 makes the race deterministic against a real
backend process (the consumer's reconnect is gated until the restarted backend has accepted frames): it fails with
v5.2's adapter and passes with v6's. L14 is unchanged.

**V52-2.** The live status, `/capabilities`, the live WebSocket batches, recordings, exports and comparison eligibility
use one classifier (`telemetry/lineage.py`): a synthetic or unknown ancestor anywhere lowers the whole stream; `unknown`
is a trust class of its own (never live, distinct from synthetic; recordings of it are stored synthetic with
`synthetic_flag_reason: "unknown-origin"`); `physical_verified` stays false.

**V52-3.** `site/tests/lib/procs.js` binds each stop to its own child: the fallback SIGKILL timer captures the child it
was created for and is cleared on exit; repeated stops return the same promise; a child that cannot be signalled
neither hangs the suite nor leaks timers or listeners.

**V52-4.** `dbtool export-audit` follows every recorded lineage and replay link transitively, read-only: confirmed
copies with their chains, unresolved ones (missing sources, ambiguity, cycles), missing intermediates, v5.2 and later
descendants of corrupted copies; corrected exports of the original and unaffected controls stay clear.

## 4. Build Your Own Cube

`#build` (rendered by `site/js/connect/build-ui.js` from `site/data/kits/` and `site/data/bom/`, generated by
`hardware/tools/make_kit.py` from `hardware/bom/aeris-kit-a1.bom.json`): the readiness ladder (concept → candidate →
engineered → manufactured → bench-tested → beginner-validated) with **A1 at candidate**; every BOM item with quantity,
status, function, the manufacturer facts with their source and check date, compatibility notes, supplier links and
tested substitutions (none); downloads only for files that exist, with their SHA-256 (the BOM, the signed firmware
manifest), and every missing asset listed with the reason (CAD, drawings, PCB, harness, photographs — none exist); the
sizing calculation (`hardware/tools/sizing.py`: with the nominal motor model the candidate driver's continuous current
holds the nominal cube only up to ≈ 1.7° from balance); guided steps with parts, expected result, checkpoint and
troubleshooting, blocked steps with the reason; progress notes kept in the browser per kit revision (never a
qualification; a revision change says what no longer carries over). The same content as one page:
`hardware/BUILD_GUIDE.md`.

## 5. Connect My Cube and the device stack

**Page** (`site/js/connect/`, loaded only when opened): choose the synthetic device (the firmware core and the models
compiled to WebAssembly, labelled SYNTHETIC — NOT A CUBE, with on-screen "hands"), a USB port (Web Serial; a port
without AERIS firmware offers only Install) or a device of your account on a backend; Install, Pair, Diagnostics,
Calibrate, Profile, Configure, Experiment, Record & compare; every page links back to the virtual cube.

**Protocol** `aeris-device-link/1` ([DEVICE_LINK.md](site/docs/DEVICE_LINK.md)): CRC-framed JSON lines, version and
capability negotiation (no substitution; read-only for unknown protocols), link sessions with heartbeats and lease
epochs, pairing with a proof signed by the device after a press of ARM on the cube, gateway authentication by the same
key, bounded commands (ids, sequence numbers, ttl from receipt, duplicate cache, status history), device state,
telemetry with resend from the device's ring and explicit gaps, the approved configuration bound by its SHA-256.

**Backend** (`/api/v6`, [DEVICES.md](site/docs/DEVICES.md), [CONNECT.md](site/docs/CONNECT.md)): owners and
authentication (`local` on loopback; `jwt` with an external identity provider; fail closed), pairing, revocation,
profiles and immutable revisions, recipes, kits, calibrations computed from the device's measurements (gyroscope bias,
IMU mounting from two or more faces, wheel directions turned by hand), approved configurations (gains designed by the
backend on the profile's model), the control lease, bounded commands, per-device telemetry continued across routes,
device recordings, Simulate My Cube (labelled by the parameter-set coverage rule from the profile's value sources) and
comparison (never admissible for synthetic data). Routes: the companion, the page relaying its Web Serial port, the
device's own outbound WebSocket (gateway).

**Safety statements, as implemented.** Release firmware never drives motors (actuation is compiled out; the website
installs only such images; an image whose manifest declares actuation is refused). An experiment starts only after a
press of ARM on the cube, runs at most its configured duration and ends with a normal stop; loss of the supervising
link ends it after `limits.link_loss_max_s`; `stop` is accepted from any session without a lease. Protection latches
FAULT and removes driver power on the local stop, tilt, speed, current, temperature, voltage, a lost or failing driver,
stale encoder estimates, a lost IMU or invalid estimate, and late control steps; clearing needs the cause gone and a
long press on the cube. There is no public one-click bypass of the missing hardware qualification: the reference
profile is `experimental`, and actuation exists only in experimental builds made from source.

## 6. Firmware

`firmware/` ([README](firmware/README.md)): the portable C11 core (4.3 k lines), the ESP32-S3 board layer (ESP-IDF
v6.0.3: I2C, TWAI, GPIO, USB-Serial-JTAG, Wi-Fi + outbound TLS WebSocket, NVS, PSA Crypto), the synthetic device's
models, the native synthetic device process, the WebAssembly build, tests and release tools.

* **Host tests** (`firmware:host`, 38): units, the estimator's and both controllers' parity with the backend's Python
  (max |Δq| 4.1e-6 rad; max |ΔT| 1.1e-6 and 1.6e-6 N·m), the link, commands, states, safety, synthetic edge and corner
  balance of the engine's plant, pairing, telemetry backfill, calibration, the release build, missing hardware,
  commissioning, update, network settings; allocation limits and the saturation report; every protection fault
  injected into the models. Negative control (`evidence/v6.0-verification/firmware-fault-tests/`): removing each
  limit, the saturation count or a protection check makes the matching test fail — **13 of 13 detected**.
* **ESP32-S3 images** (`evidence/v6.0-verification/firmware-build/`): release, experimental and qemutest built from
  clean build directories at `547fb12`, no compiler warning; the release image (1 069 872 bytes) is published with its
  signed manifest; the experimental image (actuation on) is not distributed.
* **QEMU** (`firmware:qemu`, 22 checks): the test image built from the current sources boots in the emulated ESP32-S3,
  its PSA ECDSA signature verifies with OpenSSL, it speaks the device link, accepts the approved configuration with the
  backend's SHA-256, fails its self-test naming the absent IMU and drivers (nothing fabricated), refuses an experiment,
  creates its pairing key, refuses framing errors. The emulator's timing says nothing about the board.
* **Host timing** (`evidence/v6.0-verification/firmware-timing/`): while balancing, the control step takes ≈ 2.1 µs on
  average, ≈ 13 µs at the 99th percentile and 0.55 ms at most (one step in 4000; a shared 2-CPU virtual machine) on the
  host CPU, against a 2000 µs period — not board timing.
* Observation from the saturation test: with the reference profile's nominal model, the edge controller's request
  exceeds the candidate driver's torque limit for ≈ 0.1 s after a 1° start (53 of 1501 steps) — consistent with the
  sizing result; a property of the nominal model, not a measurement.

## 7. Hardware content and what remains physical

`hardware/` ([README](hardware/README.md)): the provisional BOM (development electronics named with their published
data and check dates; motors, rotors, frame, harness, power system, local-control parts and fixture **not selected or
designed**), the sizing calculation, the kit generator, the build guide, the integration notes (pins, IMU, every CAN
driver setting the firmware relies on, local controls, drive power) and the commissioning list. **What must be done
physically before motors may be energised** — engineering (selecting motor, rotor and driver together; frame, guards,
mounts, contact pieces, fixture, harness in CAD; power system with a regenerative path; controller board; hardware stop
chain; drive-power switch; driver configuration) and measurements (mass, edge, centre of mass, inertia tensor, rotor
inertias, motor constants, friction, qualified limits, IMU noise/bias/position/orientation, wheel directions, bus
voltage under load, CAN timing, **control-loop timing on the ESP32-S3**, estimator accuracy against an independent
reference, communication-loss behaviour, normal stop, local controls) — is listed in order in
[hardware/COMMISSIONING.md](hardware/COMMISSIONING.md). Product hardening (flash and NVS encryption, secure boot, a
production signing key, a public-CA gateway certificate and an identity provider) is listed there too.

## 8. Test totals

| Run | Commit | drift | backend | firmware (host + QEMU) | JavaScript (14) | live | browser (7) | Total | Verdict |
|---|---|---:|---:|---:|---:|---:|---:|---:|---|
| full run 1 (`evidence/v6.0-verification/full-run-1-56dd89e/`) | `56dd89e` | 12 | 535 | 33 + 22 | 211 | 15 | 267 | 1095 passed | PASSED — all 26 suites; not the release's run (§12) |
| definitive run (`test-logs/`) | `91f78f3` | 12 | 540 | 38 + 22 | 211 | 15 | 267 | **1105 passed**, 0 failed, 0 skipped, 0 not run | **PASSED — all 26 suites ran and passed; every test case executed (full validation on Linux; exit 0)** |

Browser suites in the definitive run: browser_check 50, browser_v3 49, browser_v4 24, browser_v5 36, browser_v51 12,
browser_v52 45, browser_v6 51. JavaScript: assistant 9, connect 9, engine 9, numerics 38, platform 16, procs 7, resume
14, runs 14, schema 7, service 19, ui 16, v4 27, v5 17, v51 9. The QEMU suite ran with Espressif's QEMU and the
qemutest image built at `547fb12` from the same firmware sources (the suite's first check compares them). Relative to
the v5.2 runner's count (864 in 19 suites), v6.0 adds 87 backend tests, 30 JavaScript tests (connect, procs, resume),
one live test (L15), the browser_v6 suite (51) and three new suites (drift 12, firmware:host 38, firmware:qemu 22).
These counts are what this run executed; no count was used as a pass criterion. No scientific tolerance was widened;
one generated firmware fixture was regenerated with the pinned NumPy (§12); no historical fixture was changed. The
first full run is kept as evidence and is not counted in the release's totals.

## 9. Numerical reproducibility

The engine is unchanged (1.1.0). `studies/verify_campaign.py` re-ran the same 32 sampled campaign rows the reviews used
with backend 1.2.0: **32 identical** (bit for bit), 0 outcome changes, tolerances unchanged
(`evidence/v6.0-verification/numerical/`). The backend suite's reproduction and parity tests pass.

## 10. CI

**Not executed on a hosted CI service** (this project has no repository configured for this session; creating one was
not authorised). The workflow (`.github/workflows/ci.yml`) gained a `drift` job (ubuntu-24.04: the drift suite,
including the byte-identical WebAssembly rebuild with that release's clang 18) and a `firmware` job (the host tests);
the backend job installs the OpenSSL headers when missing (the end-to-end tests build the native synthetic device).
Executed instead, and **not** a hosted CI run: `scripts/ci_local.py` ran every `run:` step of the `drift`, `firmware`,
`backend`, `javascript` and `integration` jobs verbatim, each in its own fresh clone of `c607e66`: **every step
succeeded** — drift 12, firmware 38, backend 540, JavaScript 211, live 15 (`evidence/v6.0-verification/ci-local/`).
The `browser` job was not executed locally: its install step downloads a browser, which this environment must not do;
the browser suites ran in the definitive run with the preinstalled Chromium. The QEMU suite is not part of the
workflow (it needs ESP-IDF and Espressif's QEMU; documented in the workflow's header).

## 11. Release archive

`scripts/validate_release.py` now has 21 steps: the 17 of v5.2 plus **device-journey** (the device stack of the
extracted package against its freshly installed, running backend: the native synthetic device built from the package's
sources, paired through the companion only after its local press, measured, calibrated, configured, self-tested; an
experiment that waits for the local enable; a synthetic recording; Simulate My Cube; a comparison that is not
evidence; export/reimport; the gateway route continuing the telemetry session; revocation), **drift**,
**firmware-host** and **browser-v6** (with `--browser-python`). `scripts/package_release.py` now takes exactly the
tracked files of a clean commit. The candidate archive (commit `b5a2967`, 1 179 files, 13 009 829 bytes, SHA-256
`eb2608c8…c89141bf`) passed **21 of 21** steps in a fresh directory (install with Python 3.12.3 matching the lock file;
device-journey with the synthetic device built from the package; drift 12 / 12; firmware-host 38 / 38; browser-cancel
45 / 45 and 49 / 49; browser-v6 51 / 51 with Playwright 1.56.0), and the archive equals that commit file for file and
byte for byte against a verified bundle of the complete history (66 commits) (`evidence/v6.0-release/`). The delivered
archive is built from the commit tagged `aeris-v6.0` (this report's commit, which adds only `evidence/v6.0-release/`
and completes this section and the matrix's release rows); its own fresh-directory validation, its archive ↔ bundle
check and its SHA-256 are delivered next to it (they cannot be inside the archive they describe).

## 12. Problems found and corrected during the work

* The definitive test run was preceded by a full run at `56dd89e` (all 26 suites passed, kept in
  `evidence/v6.0-verification/full-run-1-56dd89e/`). Filling the acceptance matrix afterwards showed gaps in test
  coverage, corrected before the definitive run: several protection paths of the firmware were implemented but not
  exercised by a test; the allocation's saturation flags were computed but not reported; the profile's inertia check
  lacked the triangle inequality; Simulate My Cube took its run label from the backend's global parameter set instead
  of the profile's value sources; the companion had not run on a real serial device nor the gateway over TLS; the phone
  check ran at 390 px where the criterion says 375 px. Each was implemented and tested (§6, matrix rows FW-5, FW-6,
  PRF-2, PRF-5, CON-2, CON-3, BLD-5); the firmware images were rebuilt and signed again.
* The drift suite found a committed firmware test fixture generated with NumPy 2.4.4 instead of the pinned 2.5.3 (the
  corner controller torques differed by < 1e-12 N·m); regenerated with the locked environment.
* `scripts/package_release.py` would have packaged ESP-IDF build output from the working tree and left out the tracked
  review-pack ZIP; it now packages exactly the tracked files of a clean commit.
* The release manifest of the first build named the toolchain "ESP-IDF None" (the build directory's version field);
  corrected, and the manifest now also records the firmware sources' SHA-256.
* A first run of the negative control of the history probe on v5.2 failed on a missing module (only the adapter file
  had been extracted), not on the defect; re-run against v5.2's complete site scripts.

## 13. Limitations and what remains

* Everything physical (§7). No measurement was produced or claimed; every recording used here is synthetic or an
  artificial fixture; connection, pairing, flashing and close numerical agreement mark nothing VALIDATED.
* Web Serial and the installer were tested with scripted test doubles, never with a board or a USB-serial chip; the
  board's TLS client, its Wi-Fi and its CAN and I2C hardware were not exercised; ESP32-S3 timing is unmeasured.
* The firmware's allocation does not model the motor's voltage (back-EMF) limit; the driver applies it.
* A non-finite IMU value cannot be produced by the register model and is not exercised (the code refuses it).
* No production signing key, public deployment, identity provider or hosted CI run exists; the development fixture
  and the development test key are for this computer only.
* macOS, Windows and non-Chromium browsers were not run by us; accessibility was checked by automated checks only
  (headings, keyboard use of the device steps, phone widths), not with assistive technology.

## 14. How to run

```bash
scripts/install.sh --python python3.12
backend/.venv/bin/python -m pip install playwright==1.56.0 && backend/.venv/bin/python -m playwright install chromium
python3 scripts/run_tests.py --backend-python "$PWD/backend/.venv/bin/python" --browser-python "$PWD/backend/.venv/bin/python" \
        --qemu /path/to/qemu-system-xtensa --esptool-python /path/to/idf-venv/bin/python    # QEMU suite: firmware/README.md
firmware/tools/build_esp32s3.sh all                                                     # with IDF_PATH etc. set
python3 scripts/check_drift.py                                                          # with the backend's Python
cd studies && ../backend/.venv/bin/python verify_campaign.py --out /tmp/campaign-verification.json && cd ..
python3 scripts/validate_release.py --zip aeris-site-v6.0.zip --expect-top aeris-v6.0 --browser-python "$PWD/backend/.venv/bin/python"
python3 scripts/compare_archive_bundle.py --zip aeris-site-v6.0.zip --bundle aeris-site-v6.0-git.bundle --ref aeris-v6.0
backend/.venv/bin/python -m aeris_backend.tools.dbtool export-audit --db data/aeris.sqlite3     # read-only
```
