# AERIS v6.0 · source package

AERIS (Autonomous Experimental Reaction-wheel Inertial Stabilization): the website and research platform (`site/`),
the Python research backend behind its SIMULATION and HARDWARE modes and the v6.0 device API (`backend/`), the device
firmware (`firmware/`) and the reference cube's build content (`hardware/`).

**The virtual cube stays the default.** The site opens on it, and AERIS Lab — controllers, disturbances, sweeps,
analysis — still runs from `file://` with no backend, account or hardware. v6.0 adds two optional journeys:
**Build Your Own Cube** (`#build`: the reference kit A1, its provisional bill of materials with checked sources, the
sizing calculation, the guided steps and what is still missing) and **Connect My Cube** (`#connect`: signed firmware
installation over Web Serial, pairing with a local press on the cube, diagnostics, calibration, approved
configurations, bounded commands, telemetry and recordings, **Simulate My Cube** and comparison). It also corrects the
four findings of the independent review of v5.2 (V52-1 … V52-4).

**What is real.** Everything is implemented and tested in software, with **synthetic** devices (the firmware core
driving models of the IMU, the wheel drivers and the engine's plant — in the page as WebAssembly, or as a native
process) and labelled test doubles; the ESP32-S3 images build with ESP-IDF v6.0.3 and the test image runs in
Espressif's QEMU. **No physical cube has been built, connected, measured or balanced**, no firmware has run on a
physical board, and the reference kit is a *candidate* (motors, rotors, frame, harness and power system are not yet
selected or designed). All simulated numbers use nominal design parameters; nothing here is a hardware measurement.
What remains before motors may be energised: [`hardware/COMMISSIONING.md`](hardware/COMMISSIONING.md). What v6.0
changed, how it was tested, the capability table and what remains: **`SOFTWARE_COMPLETION_REPORT.md`**; acceptance
criteria and their evidence: `ACCEPTANCE_MATRIX_V6.md`. Upgrading from v5.2: `site/docs/MIGRATION_V6.md`.

## Quick start (Linux; macOS expected to work, not tested)

Needs Python 3.12+ (Node.js 18+ only for the JavaScript tests). No accounts, keys or cloud services.

```bash
scripts/install.sh                 # backend/.venv with the pinned versions (add --plots for the study figures)
scripts/start.sh --emulator        # site + API on http://127.0.0.1:8750/ ; SYNTHETIC telemetry emulator on 127.0.0.1:47100
# open http://127.0.0.1:8750/#lab      (SIMULATION appears once the backend has answered; HARDWARE opens the Telemetry tab)
# open http://127.0.0.1:8750/#connect  (the synthetic device in the page: the whole device journey, labelled SYNTHETIC)
scripts/stop.sh                    # clean shutdown; unfinished jobs resume at the next start
```

* Data (database, backups, telemetry files, logs, PID files) live in `./data` (`--data DIR` or `AERIS_DATA_DIR`).
  Back up with `backend/.venv/bin/python -m aeris_backend.tools.dbtool backup --db data/aeris.sqlite3`. A v6.0
  database (schema 3) is refused by v5.2; back up before upgrading (`site/docs/MIGRATION_V6.md`).
* Everything binds to 127.0.0.1 and the device API runs in `local` mode (one owner). A public deployment needs an
  identity provider and TLS (`AERIS_AUTH=jwt`; `site/docs/CONNECT.md` §8); none was performed.
* DEMO mode needs no backend at all: open `site/index.html` directly, or serve `site/` with any static server. The
  Build page and the synthetic device in Connect My Cube work there too.
* A device of your account on the backend: the companion (`python -m aeris_backend.companion`), the page relaying a
  Web Serial port, or the device's own outbound connection; the native synthetic device stands in for a board
  (`site/docs/CONNECT.md` §7).
* Windows (not tested): create the venv with `py -3.12 -m venv backend\.venv`, then
  `backend\.venv\Scripts\pip install -r backend\requirements-lock.txt` and `pip install --no-deps -e backend`, and
  run `backend\.venv\Scripts\python -m aeris_backend.api.main --serve-site site` from the repository root.

**Troubleshooting.** `scripts/start.sh` prints the last log lines if the backend exits (`data/logs/backend.log`).
"The database is in use by another AERIS backend process" means another backend holds that database (one process
per database; stop it or use `--data`). A port in use: `--port`. SIMULATION stays disabled until `/api/v1/health`
answers ("Check backend" re-probes). The device API's state and the reason when it is disabled: `/api/v6/status`.

## Firmware and hardware

* `firmware/` — the portable C11 core (states, sensing and estimation, the edge and corner controllers, allocation,
  protection, the device link `aeris-device-link/1`, bounded commands, telemetry), the ESP32-S3 board layer, the
  synthetic device's models and the tests. Host build: `make -C firmware test`. ESP32-S3: ESP-IDF v6.0.3
  (`firmware/tools/build_esp32s3.sh`). [`firmware/README.md`](firmware/README.md).
* The website installs only the signed **release** image (`site/firmware/releases/0.1.0/`), which never drives motors.
  It is signed with a development test key; there is no production key. Motor actuation exists only in experimental
  builds made from source by the person responsible for a qualified cube in a fixture
  ([`site/docs/FIRMWARE_INSTALL.md`](site/docs/FIRMWARE_INSTALL.md)).
* `hardware/` — the bill of materials (`bom/`), the kit generator, the sizing calculation, the build guide, the
  integration notes (pins, IMU, CAN, local controls, power) and the commissioning list.
  [`hardware/README.md`](hardware/README.md).

## Tests

```bash
python3 scripts/run_tests.py       # every suite; logs, exact counts, summary.json and SUMMARY.md in test-logs/
python3 scripts/run_tests.py --only backend,js      # a selection: reported as "selected suites only", never a full validation
```

Suites: **drift** (every generated or built file matches its generator or build: `index.html`, schemas, kit and site
data, sizing block, fixtures, the signed firmware release's source hash, the WebAssembly device and its byte-identical
rebuild with the recorded compiler, vendored files); **backend** pytest (engine, parity, API, jobs and recovery,
persistence, telemetry, provenance, export audit, the device API, authentication, pairing, leases, commands,
calibration, configurations, telemetry continuity, the native synthetic device end to end, the test runner itself);
**firmware:host** (the firmware core's host tests: C, needs make, a C compiler and OpenSSL's libcrypto);
**firmware:qemu** (the ESP32-S3 test image in Espressif's QEMU: needs the qemutest build, `qemu-system-xtensa` and a
Python with esptool — `--qemu`, `--qemu-build`, `--esptool-python`); the JavaScript suites (schemas, numerics,
platform, runs, service, UI, v4, v5, v51, assistant, engine, connect, procs, resume); the live adapter integration
(the site's real adapters against a started backend); and the Playwright browser suites (reduced motion, bounded
waits; `browser_v6` covers the virtual-first site, Build, Connect with the synthetic device, a scripted Web Serial
port and flasher, a local and a jwt-mode backend). Interpreters and tools given explicitly are never silently
replaced; a suite that cannot run is reported NOT RUN with the reason. Skipped test CASES are listed by name and
never counted as passed. Exit codes: 0 every selected test case executed and passed (with every suite selected: a
full validation); 1 a failure; 2 incomplete (a suite or test case did not run); 3 passed with a declared platform
exclusion (named; not a full validation); 64 a usage error. On a machine without ESP-IDF and Espressif's QEMU the
suite `firmware:qemu` is NOT RUN, so the run is INCOMPLETE — never a full validation.

CI runs the drift, backend, firmware host, JavaScript, live and browser suites (`.github/workflows/ci.yml`; not the
QEMU suite); `python3 scripts/ci_local.py --python python3.12` executes its steps locally from clean clones (that is
not a CI service run). A release archive is built with
`python3 scripts/package_release.py --top aeris-v6.0 --out aeris-site-v6.0.zip` and checked end to end in a fresh
directory with `python3 scripts/validate_release.py --zip aeris-site-v6.0.zip --expect-top aeris-v6.0
[--browser-python PY]` (install, start, the v5.x research and telemetry steps, the v6.0 device journey with the
synthetic device built from the package, drift, firmware host tests, the test runner, the browser suites, backup,
stop; JSON result).

## Run on macOS (Apple silicon or Intel)

Python 3.12+ (python.org installer or `brew install python@3.12`); Node.js 18+ only for the JavaScript tests
(`brew install node`); the Xcode command-line tools and OpenSSL (`brew install openssl@3`) for the firmware host tests
and the native synthetic device. From the unpacked `aeris-v6.0/` folder:

```bash
scripts/install.sh --python python3.12          # backend/.venv with the pinned versions
scripts/start.sh --emulator                     # http://127.0.0.1:8750/#lab ; SYNTHETIC emulator on 127.0.0.1:47100
scripts/stop.sh
backend/.venv/bin/python -m pip install playwright==1.56.0 && backend/.venv/bin/python -m playwright install chromium
python3 scripts/run_tests.py --backend-python "$PWD/backend/.venv/bin/python" --browser-python "$PWD/backend/.venv/bin/python"
```

On macOS the Linux `/proc` orphan-worker test is skipped and accepted only as the one declared platform exclusion
(`scripts/platform_exclusions.json`) while its portable `ps` counterpart passes (exit code 3, never a full
validation). The firmware's Makefile expects OpenSSL's headers and libcrypto on the compiler's default search paths,
which Homebrew's OpenSSL is not on by default (not tested). v6.0 was built and tested on Linux x86-64; the macOS
steps were not executed by us.

## Layout

- `site/` — the website (static; DEMO works offline). `site/README.md`. v6.0: `js/connect/` (Build and Connect,
  loaded only when opened), `data/` (kits, BOM, reference profile, recipes, approved reference configurations),
  `firmware/` (the signed release, the trust list, the WebAssembly synthetic device), `vendor/` (esptool-js, @noble).
- `backend/` — the research backend (Python 3.12+, FastAPI) and the device API `/api/v6`. `backend/README.md`.
- `firmware/` — device firmware, synthetic device, tests, build and release tools. `firmware/README.md`.
- `hardware/` — BOM, kit generator, sizing, build guide, integration, commissioning. `hardware/README.md`.
- `site/docs/` — PHYSICS, CONTROLLERS, ESTIMATION, SCHEMA_EXTENSIONS, BACKEND_INTEGRATION, RESEARCH_PLATFORM,
  EXPERIMENT_CONTRACT, METRICS, ESTIMATION_TIMING, RECORDING, JOBS_AND_RECOVERY, DATABASE, PARAMETERS,
  REPRODUCIBILITY, TELEMETRY, HARDWARE_HANDOFF, HARDWARE_NEXT_STEPS, ENGINE_AUDIT; v6.0: CONNECT, DEVICES,
  DEVICE_LINK, FIRMWARE_INSTALL; migrations MIGRATION_V5, _V5_1, _V5_2, _V6.
- `site/schemas/` — JSON Schemas for the research file formats (generated: `node build/build_schemas.js`); the device
  documents' schemas are in `backend/aeris_backend/devices/schemas.py` (DEVICES.md).
- `build/` — sources of `site/index.html` (`python3 build/build_index.py`) and the single-page artifact build.
- `studies/` — the v5 nominal simulation campaign, its reproduction check, solver verification and figures.
- `backend/benchmarks/` — software speed benchmarks.
- `evidence/` — reproductions of every review's findings on the unmodified release and re-runs on the next one;
  v6.0: `v6.0-inputs/` (the specification and blueprint as supplied), `v6.0-verification/` (V52-1…4 before and after,
  firmware builds, host timing, the numerical check), `v5.2-independent-review/` (the review as supplied),
  `v5.2-release/` (the v5.2 test run).
- `scripts/` — install, start, stop, run_tests (with `platform_exclusions.json`), check_drift, ci_local,
  package_release, validate_release, compare_archive_bundle.
- `SOFTWARE_COMPLETION_REPORT.md` (v6.0), `ACCEPTANCE_MATRIX_V6.md`, `CHANGELOG.md`; earlier reports
  `V5_2_SOFTWARE_COMPLETION_REPORT.md`, `V5_1_SOFTWARE_COMPLETION_REPORT.md`, `V5_SOFTWARE_COMPLETION_REPORT.md`,
  `V4_REPORT.md`, `V4_PLAN.md`.
