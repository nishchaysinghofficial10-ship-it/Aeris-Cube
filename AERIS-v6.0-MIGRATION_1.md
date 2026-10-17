# Migrating from v5.2 to v6.0

v6.0 keeps the **virtual cube as the default** — the site still opens on it and AERIS Lab, the controllers, the
disturbances, the sweeps and the analysis still run from `file://` with no backend, account or hardware — and adds an
optional **Build Your Own Cube** journey and **Connect My Cube** (firmware, pairing, bounded commands, calibration,
approved configurations, device telemetry and recordings, Simulate My Cube, comparison). It corrects the four
findings of the independent review of v5.2 (V52-1 … V52-4).

**Unchanged:** the research engine (`aeris-research-engine 1.1.0`) and its numerical behaviour (the sampled campaign
rows reproduce within the unchanged tolerances), stored records and their schemas, the `aeris-telemetry-ndjson/1`
contract, the `/api/v1` contract in local mode, the `LocalSimulatorAdapter` contract, the frozen v5.2 release
artifacts. The backend is **1.2.0** (`aeris-research-backend`). Nothing stored is rewritten.

## 1. Before you upgrade

```bash
backend/.venv/bin/python -m aeris_backend.tools.dbtool backup --db data/aeris.sqlite3     # a verified copy; nothing overwritten
```

A v5.2 backend refuses a database written by v6.0 (schema 3: "written by a newer AERIS backend; it is not
modified") — keep the backup if you may go back.

## 2. Install and launch

`scripts/install.sh` installs the pinned dependencies of `backend/requirements-lock.txt`; v6.0 adds `cryptography`
(ECDSA P-256 verification of device signatures; with `cffi` and `pycparser`), `PyJWT` (bearer tokens in jwt mode) and
`pyserial` (the companion's serial port; `websockets` was already pinned). The companion's two dependencies are also
declared as the package's optional `companion` extra for installs without the lock file.
`scripts/start.sh` is unchanged (backend + site on `http://127.0.0.1:8750/`, `AERIS_AUTH=local`). Local launch, the companion and the development
identity fixture: [CONNECT.md](CONNECT.md) §7; a public deployment: CONNECT.md §8 (none was performed).

## 3. Database (schema 2 → 3, additive)

The migration runs at start-up and is logged in the `migrations` table (`dbtool verify` shows it):

| Change | Purpose |
|---|---|
| table `devices` (id, owner, name, public key, board, firmware, synthetic flag, paired, revoked, profile id/revision, last seen) | paired devices and their credentials |
| table `pairings` (session, owner, nonce, created, expires, used, device, outcome) | single-use pairing sessions |
| table `leases` (device, owner, holder, epoch, acquired, expires) | one control lease per device |
| table `commands` (id, device, owner, holder, action, params, config hash, created, ttl, status, history, result, route, lease epoch) | the command log |
| column `runs.owner` | twin runs belong to the owner of the profile they simulate (jwt mode) |
| document collections `profiles`, `profile_heads`, `recipes`, `calibrations`, `device_configs` in the existing `docs` table | immutable device documents ([DEVICES.md](DEVICES.md)) |

`dbtool backup`, `restore` and `verify` cover the new tables; a restore of an older backup migrates the restored copy
(the backup file is not changed). At start-up, commands that never reached a final status are marked `expired` (never
resent).

## 4. API

* **`/api/v1` in local mode** — unchanged, plus additive fields: `/capabilities` has a `devices` section (`/api/v6`,
  its authentication summary); a twin run carries `twin {profile, recipe, calibration, owner}`; a device recording
  carries `device_link {device_id, config_hash, boot_id, route}`; the telemetry status and every live batch's
  `provenance` may now say `trust: "unknown"` (§5).
* **`/api/v1` in jwt mode** (`AERIS_AUTH=jwt`, new) — every request except `/health` and `/capabilities` needs
  `Authorization: Bearer <token>`; owned records (device recordings, twin runs, datasets made in jwt mode, their jobs)
  are visible to their owner only (404 for others); v1 live sockets present a ticket (`POST /api/v6/tickets
  {"purpose": "v1-live"}`); the global v1 telemetry bridge is switched off (each device has its own hub in `/api/v6`).
* **`/api/v6`** (new) — devices, pairing, profiles, recipes, kits, calibrations, approved configurations, leases,
  bounded commands, per-device telemetry and recordings, the twin: [DEVICES.md](DEVICES.md) §7. A twin run's
  `validation.parameter_set` is `profile:<id>:r<revision>` (its label follows the profile's value sources through the
  parameter-set coverage rule; DEVICES.md §6), never the backend's global parameter set. In local mode it is
  offered only on a loopback bind; with a non-loopback bind or a public origin it stays disabled unless jwt mode is
  configured completely (fail closed, the reason in `/api/v6/status`).
* The error envelope is unchanged (`{"error": {"code", "message", "details", "retryable"}}`), now shared by v1 and v6.

## 5. Telemetry clients and provenance (V52-1, V52-2)

* **Race-safe resume (V52-1).** `HardwareTelemetryAdapter.live` no longer advances past frames it never received: on
  every (re)connection and stream change it reads the backend's retained history for the missing range, holds the live
  frames until the history is in, and delivers each frame once and in order. Two new events: `{"type": "catchup",
  "state": "started" | "done"}` and `{"type": "gap", "stream", "from", "to", "missed", "reason"}` when frames are no
  longer retained (the cursor moves past them only then). A page that consumes `live()` should show gaps rather than
  ignore them (the Lab's Telemetry tab does). No backend change is needed beyond what v5.1/v5.2 already send.
* **One conservative live-provenance rule (V52-2).** A live hello that declares `physical` but whose lineage names a
  synthetic or unknown ancestor is no longer reported as a live physical device. `unknown` is now its own class in the
  live status (`trust: "unknown"`, `synthetic: false`, `declared_physical: false`, never `live`), distinct from
  `synthetic`; a client that tested only `synthetic` must also handle `trust == "unknown"`. Recordings of an
  unknown-origin source are stored with `synthetic: true` and `provenance.synthetic_flag_reason: "unknown-origin"`
  (never evidence). [TELEMETRY.md](TELEMETRY.md) §6 and §12.
* **Test helper (V52-3)** — affects only the test suites: `site/tests/lib/procs.js` binds every stop timer to its own
  child process.

## 6. Legacy export audit (V52-4)

v5 and v5.1 exported NDJSON without converting units (v5.2 fixed the export, MIGRATION_V5_2.md); recordings made from
such exports hold raw numbers under wire-unit labels. v5.2's audit looked only one generation deep. v6.0's audit
follows every recorded lineage and replay link **transitively** and reads the database **read-only**:

```bash
backend/.venv/bin/python -m aeris_backend.tools.dbtool export-audit --db data/aeris.sqlite3 > export-audit.json
```

The answer (`aeris-export-audit/2`) lists:

| Key | Meaning |
|---|---|
| `originals_whose_v5_v51_ndjson_export_was_wrong` | originals stored in non-wire units (correct and unchanged themselves) |
| `affected` | **confirmed** corrupted copies, any number of generations deep and through any later export version, each with its `chain` to the original, how it was decided, and intermediates missing from the database |
| `unresolved` | copies whose chain cannot be decided (a missing source, an ambiguous reference, a cycle, an unresolved parent) — never reported as clear |
| `clear_copies` | the number of copies whose every link was followed without reaching corrupted data |
| `copies_recorded_from_such_an_export` | the v5.2 key, now transitive |
| `cycles`, `limits`, `what_to_do` | — |

Affected copies cannot be repaired (their original units are not recorded in them): export the **original** again
with v5.2 or later and record that replay instead; check each unresolved copy against its source by hand. The audit
changes nothing (the database is byte-identical before and after; `backend/tests/test_export_audit.py`).

## 7. Documents and protocols (new)

| Identifier | What | Reference |
|---|---|---|
| `aeris-device-profile/1`, `aeris-experiment-recipe/1`, `aeris-calibration/1` | what a cube is, what an experiment asks for, what was measured | [DEVICES.md](DEVICES.md) |
| `aeris-device-config/1`, `aeris-plant-model/1` | the approved configuration; the synthetic device's plant | [DEVICE_LINK.md](DEVICE_LINK.md) §8 |
| `aeris-kit/1`, `aeris-bom/1` | kit revisions and the bill of materials | DEVICES.md §5, [`hardware/README.md`](../../hardware/README.md) |
| `aeris-firmware-manifest/1`, `aeris-firmware-trust/1` | signed firmware releases and the installer's trust list | [FIRMWARE_INSTALL.md](FIRMWARE_INSTALL.md) |
| `aeris-device-link/1` | the host ↔ firmware protocol | [DEVICE_LINK.md](DEVICE_LINK.md) |
| `aeris-telemetry-ndjson/1` | unchanged; the firmware's telemetry hello adds `session`, `boot_id`, `config_hash` and `attitude` (additional fields, ignored by older readers) | [TELEMETRY.md](TELEMETRY.md) |

## 8. Site

New pages `#build` and `#connect` (navigation, hero entry points and footer links; "Try the Virtual Cube" stays the
primary action), `css/connect.css`, `js/connect/*` (loaded only when one of the pages is opened), `data/` (kits, BOM,
reference profile, recipes, approved reference configurations; `data/aeris-data.js` for `file://` pages),
`firmware/` (the release manifest and images, the trust list, the WebAssembly synthetic device), `vendor/`
(`esptool-js` 0.7.0, `@noble` P-256/SHA-256; `vendor/VENDORED.md`). The page templates gained `build.html` and
`connect.html` (`build/build_index.py` regenerates `index.html`). Saved Build-page progress lives in the browser and is
keyed by kit revision; it never permits anything.

## 9. Firmware and hardware (new)

`firmware/` (portable core, ESP32-S3 board layer, synthetic device, tests, release tools; [`firmware/README.md`](../../firmware/README.md))
and `hardware/` (BOM, sizing, kit generator, build guide, integration, commissioning). The website installs only the
release image, which never drives motors; actuation exists only in experimental builds made from source. Nothing has
run on a physical board.
