# Commissioning — what remains before motors may be energised (v6.0)

**Nothing physical exists yet.** No AERIS A1 cube has been designed in CAD, assembled, wired, powered, measured or
balanced; no motor has been selected; no firmware has run on a physical board. The software for the whole journey is
implemented and tested with synthetic devices (`site/docs/CONNECT.md`). This document is the specific list of physical
engineering work and measurements that remain, and the order in which the software's own steps are meant to be used
once the hardware exists.

> **Saved instructional checkboxes on the Build page are progress markers, not hardware qualification or permission
> to energise motors.** The device, its approved configuration, its self-test and a press of ARM on the cube decide,
> independently of anything ticked in a browser.
>
> **Connection, pairing, successful flashing and close numerical agreement with the simulation do not make anything
> VALIDATED.** Synthetic, replayed or undeclared data is never validation evidence; a declared-physical recording is
> evidence only under a predeclared validation protocol, reviewed by a person.

## 1. Engineering work still required

**Selection and mechanics**

- [ ] Select the **motor, rotor and driver together** with `hardware/tools/sizing.py` (SIZING.md): with the nominal
      motor model the candidate driver's continuous current holds the nominal cube only up to ≈1.7° from balance.
- [ ] Design the **frame, guards, motor mounts, IMU and controller mounts and the edge/corner contact pieces** (no CAD
      exists for A1-candidate): edge length, mass and centre of mass consistent with the profile; guards that keep
      fingers clear of the rotors; contact pieces that do not slip in the fixture.
- [ ] Design the **rotors**: inertia chosen with the motor, captive retention to the shaft, balance tolerance, a tested
      maximum speed with a margin above the configuration's wheel-speed abort limit, containment by the guard.
- [ ] Design the **encoder magnet** holder for the selected motor shaft and the driver's onboard encoder.
- [ ] Design the **commissioning fixture**: it lets the cube rotate a few degrees about its edge or corner, catches it
      before a face reaches the floor (the synthetic plant's catch is at 20° from balance — the real travel must be
      chosen for the real cube), tolerates residual wheel spin after a stop and keeps hands clear.
- [ ] Design the **harness**: keyed, labelled, motor-power wiring separated from sensor and bus wiring, mating
      connectors for the drivers' headers; lengths and fasteners follow the CAD.

**Electronics and power**

- [ ] Design the **power system**: a bus voltage inside every driver's range over the whole discharge curve (the
      reference profile's 9.9–12.6 V starts below the candidate driver's 10 V minimum); a **regenerative-energy path**
      (the candidate driver has no brake-resistor control: a battery allowed to absorb the current or an external
      regeneration clamp); fuse; main switch; a 5 V regulator for the controller; a complete protected pack and charger.
- [ ] Design the **controller board**: the CAN transceiver, regulators, connectors, the local controls; respect the
      DevKitC's mutually exclusive power inputs while USB is connected for the link.
- [ ] Build the **hardware emergency stop**: a latching switch whose power contact interrupts the motor bus
      independently of the firmware, and whose second contact keeps the stop-sense line (GPIO11) closed while running
      (open = stop).
- [ ] Build the **drive-power switch** driven by DRIVE_EN (GPIO14): relay or contactor with pre-charge, in series with
      the stop chain, off by default, off when the controller is unpowered, resetting or browning out.
- [ ] Decide the **drivers' logic supply** (INTEGRATION.md §3): if the drivers are powered only through DRIVE_EN, a
      release build never sees them on CAN and cannot pass its driver checks.
- [ ] Configure every driver (INTEGRATION.md §CAN): node ids, 1 Mbit/s, cyclic heartbeat and encoder estimates,
      answers to the RTR requests, motor/encoder calibration, torque constant, limits, **CAN watchdog**, over-voltage and
      regenerative-current limits, direction convention.

## 2. Measurements required

Each measured value enters a **new profile revision** with its `value_sources` group set to `measured` (DEVICES.md), or
a calibration. Nothing is promoted automatically.

| Quantity | Where it goes | How (suggested) | Why it matters |
|---|---|---|---|
| Body mass | `body.mass_kg` | scale | gravity torque, inertia, gains |
| Edge length | `body.edge_m` | caliper | geometry of the plant |
| Centre of mass | `body.com_m` (or `body.components`) | balance-point or suspension method | m·g·l: the torque the wheels must hold |
| Body inertia tensor | `body.inertia_kgm2` | pendulum / torsion methods about several axes | dynamics; the gains are designed on it |
| Rotor inertia J_w (each wheel) | `wheels[].Jw_kgm2` | spin-down with a known torque, or a torsion pendulum, after manufacture | momentum capacity, saturation time |
| Motor constants (each) | `wheels[].motor` (KV, R) | back-EMF at a known speed; phase resistance | torque per amp; the driver's torque constant must match |
| Friction | `wheels[].friction` (cv, cc) | spin-down curves | allocation and the model's wheel dynamics |
| Current, torque, speed and temperature limits | `wheels[].motor`, `electrical` | bench tests under the thermal duty of balancing (continuous torque near equilibrium) | qualified limits replace the nominal ones; the hard caps are only caps |
| IMU noise and bias | `imu.noise`; calibration | `calibrate_imu` repeated over time and temperature | estimator settings; gyroscope bias |
| IMU position | `imu.position_m` | measured from the pivot corner | estimator compensation |
| IMU orientation | calibration (`to_body`) | `calibrate_axes` on two or more faces | the body frame of every measurement |
| Wheel directions | calibration (`wheels[]`) | `check_encoders`, each wheel turned by hand in its marked direction | a wrong sign makes the controller push the cube over |
| Bus voltage under load and during braking | `electrical` | measured on the bench | protection limits; the regenerative path |
| CAN timing | — | `diagnostics`: heartbeat and encoder ages, `tx_errors`, under full bus load | the 25 ms encoder freshness the firmware requires |
| **Control-loop timing on the ESP32-S3** | — | `diagnostics`: `loop.step_us_max`, `step_us_mean`, `overruns` at 500 Hz, with Wi-Fi active and idle | host timing (`make -C firmware timing`) does not transfer; the self-test fails on any overrun |
| Estimator accuracy | — | the device's estimate against an **independent reference** (a known tilt on a jig, an external measurement) | six-axis data gives no absolute yaw reference; a simulated impulse is not evidence of a physical one |
| Communication-loss behaviour | — | in the fixture: stop the supervising host; the experiment must end with a normal stop after `limits.link_loss_max_s` | the policy is tested only on the synthetic plant |
| Normal stop | `limits.stop_torque_Nm` | in the fixture: wheels braked within the stop timeout, body reaction bounded, the fixture catches the cube | residual wheel spin and the fall |
| Local controls | — | `diagnostics` inputs: ARM, stop (an open wire must read "asserted"), dead-man; DRIVE_EN off at reset and brown-out | the local enable and the stop path |

A USB-tethered trial adds cable forces that the engine's untethered model does not contain: a comparison of a tethered
trial with a simulation must say so.

## 3. Order of the first powered work (once §1 exists)

1. **Release firmware**, no motor power: install (FIRMWARE_INSTALL.md), pair, diagnostics; the IMU on the bench:
   `calibrate_imu`, `calibrate_axes`; the local controls read correctly; the stop chain cuts the motor bus.
2. **Profile revision** with the measured values; `sizing.py --profile` again; a calibration for that revision.
3. **Commissioning build** (experimental, from source — §4), cube in the fixture, hardware stop wired and tested:
   per-motor tests one wheel at a time at the build's caps; direction and response checked; releasing the dead-man
   control ends each test at once.
4. `check_encoders` (each wheel by hand) until all three directions are confirmed.
5. An approved configuration built by the backend for the profile revision, the edge recipe and the calibration;
   `apply_config`; `self_test` → READY.
6. **Simulate My Cube** with the same profile revision, recipe and calibration: the predicted recovery and wheel
   saturation of the first trial.
7. **Edge trial** in the fixture: `start_experiment`, ARM pressed on the cube, at most the recipe's duration, a normal
   stop; the recording stored with its provenance; compared with the simulation (a validation claim needs a predeclared
   protocol).
8. The **corner trial** only after an edge trial of the same device and profile revision completed its duration.

## 4. The commissioning mode, as implemented (never performed)

* Only in an **experimental build** (`CONFIG_AERIS_ACTUATION_ENABLED=y`, `CONFIG_AERIS_COMMISSIONING=y`), built from
  source by the person responsible for the hardware (`firmware/README.md`). The website never installs it and has no
  motor console.
* `commission {"enter": true}` — **USB only**, from DISARMED, with an applied configuration (wheel nodes, signs and
  torque limits come from it), the **dead-man control held**, the stop released, and no other local confirmation
  pending → COMMISSIONING. Releasing the dead-man control returns to DISARMED with the drivers idle.
* `motor_test {"wheel": 0…2, "torque_Nm", "duration_ms"}` — USB only, in COMMISSIONING, the wheel's driver seen on CAN:
  torque on **one wheel**, at most the build's commissioning torque cap (**50 mN·m** by default) and the wheel's
  configured torque limit, for at most the build's time cap (**2000 ms**); it ends at its duration, at the build's
  speed cap (**600 rpm**), when the dead-man control is released, or on `stop`. Result: `{node, applied_s,
  omega_max_rads, ended}`. Between tests the drivers are idle.
* The attitude-related protections do not apply to a motor test; the electrical, driver, temperature, speed and stop
  protections do. The build's hard caps (10 A, 1 N·m, 8000 rpm) still bound every configuration.

No motor test has been run: the mode is tested only against the synthetic device's driver and plant models.

## 5. Product hardening (before any deployed product)

- [ ] **Flash encryption and NVS encryption**: the pairing key's private scalar and the Wi-Fi credentials are stored in
      NVS; without encryption anyone with physical access to the flash can read them.
- [ ] **Secure boot** (signed bootloader and application): not enabled in the v6.0 images.
- [ ] A **production firmware signing key** held outside the repository (offline or in an HSM), its public key in the
      trust list's production entries, and a release procedure; v6.0 has none (FIRMWARE_INSTALL.md).
- [ ] A gateway with a certificate from a public CA (the board verifies the server against the ESP-IDF CA bundle) and
      an identity provider for the backend (`AERIS_AUTH=jwt`, CONNECT.md § Public deployment).

## 6. Readiness

The kit stays `candidate` until §1 is done and the design is complete (`engineered`); a cube assembled to that design
with its measured profile is `manufactured`; §2–§3 completed with recorded trials is `bench-tested`; a build by someone
outside the project from the published material alone is `beginner-validated` (README.md).
