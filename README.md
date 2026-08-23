# Brinkley Home Assistant — Direct OneControl CAN Integration

Home Assistant / ESPHome integration for a Brinkley RV using an ESP32 connected directly to the coach's Lippert / LCI OneControl IDS-CAN network.

This repository documents the hardware, CAN-bus behavior, ESPHome configuration, discovered devices, calibration, safety work, and controller diagnostics used on the development coach (Berta).

> **Important:** OneControl node addresses are coach-specific. Do **not** blindly copy the node addresses in this repository to another RV. Discover and verify the devices on your own coach before enabling control.

## What the ESP32 does

The ESP32 acts as a bridge between Home Assistant and the RV's existing OneControl CAN network. It is not replacing the Lippert controller or factory panel.

The ESP32:

- joins Home Assistant through ESPHome;
- listens to OneControl CAN traffic;
- exposes OneControl devices as Home Assistant entities;
- reads tank levels and device status from the coach;
- controls verified lights and relays by transmitting addressed OneControl commands;
- controls the two tested awning motors using a remote-control session and repeated motor keepalive frames;
- watches motor-load telemetry during awning retraction;
- keeps an estimated awning position so a partially extended awning does not receive a full-extension run time again;
- watches factory-panel awning commands so the estimated position can remain synchronized when someone operates an awning from the original panel;
- can expose persistent ESP32/network/CAN diagnostics to help diagnose intermittent controller dropouts.

The ESP32 is therefore a **participant on the CAN bus**, not merely a passive CAN sniffer. Treat configuration changes accordingly.

## Currently mapped on the development coach

### Lights

- Kitchen Pendants — `0xCB`
- Island Light — `0xC9`
- Awning Light 1 — `0x3A`
- Awning Light 2 — `0x88`

### Relays

- Water Pump — `0x58`
- Tank Heater — `0x0D`

### Tanks

- Fresh Tank — `0x41`
- Grey Tank 1 — `0xE9`
- Grey Tank 2 — `0x43`
- Grey Tank 3 — `0x22`
- Black Tank — `0xCE`

### Awnings

- Awning 1 motor node — `0xCF`
- Awning 2 motor node — `0xD6`

These addresses are documented because they are useful development evidence. **They are not universal Brinkley addresses.**

## CAN hardware

The development hardware uses an ESP32 with an external CAN transceiver. The current ESPHome configuration uses:

```yaml
can_tx_pin: GPIO5
can_rx_pin: GPIO4
```

`GPIO5` and `GPIO4` are the ESP32-side TX/RX signals to the CAN transceiver. The transceiver then connects to the coach as **CAN-H** and **CAN-L**.

The OneControl bus observed on the development coach operates at **250 kbit/s**.

### CAN termination matters

A CAN network should normally have two 120-ohm terminating resistors, one at each physical end. Do not casually add another terminator because the ESP32 was added. If your connection replaces an existing terminating plug or becomes a physical endpoint, termination may be required; if it is a mid-bus tap and the original two terminators remain, adding a third terminator is wrong.

Power the ESP32/transceiver appropriately for the hardware you use. Do not assume CAN-H/CAN-L provide board power.

## ESPHome file

The main configuration is:

`esphome/brinkley-can.yaml`

It uses Andrew Fitzpatrick's `esphome-onecontrol` package pinned to `v1.2` for the OneControl core, lights, relays, tanks, CAN configuration, and session behavior.

Before compiling, create your ESPHome secrets and review every node address.

## Controller diagnostics

The optional diagnostic package is:

`esphome/diagnostics.yaml`

Add it to the main YAML under the existing `packages:` block:

```yaml
  diagnostics: !include diagnostics.yaml
```

It adds ESP uptime, persistent boot count, reset reason, Wi-Fi/API health, heap and loop timing, CAN frame counters, CAN receive rate, and time since the last received CAN frame. It is observational only and does not transmit CAN traffic or alter awning logic.

The purpose is to distinguish three otherwise similar-looking failures:

1. **ESP32 reboot/brownout/watchdog** — uptime resets and boot count increments.
2. **Wi-Fi/API/Home Assistant connectivity loss** — entities disappear without a corresponding reboot.
3. **CAN-side failure** — ESP/API remain alive while CAN receive counters stop.

See `docs/DIAGNOSTICS.md` for the entity list and failure interpretation guide.

## User calibration — easy place to tune awnings

The awning values that owners are most likely to change are intentionally grouped together in a clearly marked `USER CALIBRATION - AWNING TIMERS / SAFETY` block near the top of the YAML.

The current development-coach values are:

```yaml
awning_1_full_extend_ms: "21500"
awning_2_full_extend_ms: "22500"
awning_1_retract_stall_threshold: "3000"
awning_2_retract_stall_threshold: "3300"
```

### Full-extension time

`awning_X_full_extend_ms` is the total motor run time, in milliseconds, from fully retracted to the desired fully extended position.

Example: `21500` = 21.5 seconds.

Calibrate conservatively. Start shorter than required, test physically, and increase in small steps. Some awnings can continue past the desired extension point and begin rolling the fabric in the opposite direction, so blindly using somebody else's timer is a spectacularly bad calibration method.

### Retract stall threshold

`awning_X_retract_stall_threshold` is the motor-load value used to recognize that the awning has reached its fully retracted mechanical endpoint. The implementation requires repeated high-load observations rather than stopping on a single transient spike.

Set this from **your own motor-load telemetry**. A value that is too low can stop the awning short. A value that is too high can unnecessarily hold the motor against the closed stop.

## Awning position tracking

The awnings do not provide a simple absolute 0–100% position value, so position is estimated by motor run time.

When Home Assistant extends an awning, elapsed extension time is accumulated. If it is stopped after five seconds, that five seconds remains part of the position estimate. A second Extend command therefore runs only the estimated remaining time instead of starting another complete full-extension timer.

Retraction subtracts elapsed motor time from the estimate. A confirmed fully retracted motor-load event resets the position to `0%`, which acts as a physical calibration reference.

The configuration also watches the factory-panel motor command stream. If someone moves an awning from the original OneControl panel, the ESP32 observes that movement and updates the same position estimate. This matters for automation: Home Assistant should not assume an awning is retracted merely because Home Assistant was not the device that extended it.

That synchronization is especially important for planned safety automation such as automatically retracting awnings when an anemometer reports excessive wind.

## Home Assistant entities

ESPHome exposes the mapped lights, switches, tanks and awning controls directly to Home Assistant. The awning implementation adds controls and diagnostics including:

- Awning 1 Extend / Retract / Stop
- Awning 2 Extend / Retract / Stop
- awning state
- motor load
- estimated position
- estimated remaining extension time

## First installation procedure

1. Install ESPHome in Home Assistant or use the ESPHome CLI.
2. Connect the ESP32 to the CAN transceiver using the correct TX/RX pins for your hardware.
3. Connect the transceiver to CAN-H/CAN-L with the coach powered down while making wiring changes.
4. Verify CAN termination for your particular tap location.
5. Discover and verify the OneControl node addresses on **your** coach.
6. Copy `esphome/brinkley-can.yaml` into ESPHome.
7. Configure Wi-Fi credentials through ESPHome secrets rather than committing passwords to GitHub.
8. Change the device addresses to match your coach.
9. Review the awning calibration block before enabling motor control.
10. Optionally add `diagnostics: !include diagnostics.yaml` under `packages:`.
11. Compile and flash the ESP32.
12. Verify receive traffic in ESPHome logs before commanding anything.
13. Test low-risk devices such as a known light first.
14. Test awnings physically while standing where you can see them and with Stop immediately available.
15. Calibrate each awning's extension time and retract stall threshold individually.

## Safety notes

The RV CAN bus may also contain slides, levelers and other motorized systems. A wrong OneControl node address can therefore be much more interesting than a light failing to turn on.

- Never guess a node address.
- Never assume addresses from another coach are identical.
- Verify a device before transmitting to it.
- Do not use awning automation until travel times and retract endpoints are calibrated.
- Keep physical/manual controls available while commissioning.
- Treat wind-triggered automatic retraction as a later safety feature that requires independent validation of the wind sensor, thresholds, failure behavior and awning state tracking.

## Credits and upstream work

This project stands on excellent reverse-engineering work done by other developers.

### Andrew Fitzpatrick — `esphome-onecontrol`

Andrew's ESPHome project provides the core Lippert / OneControl IDS-CAN implementation used here: CAN setup, session handshake, device packages, discovery concepts, light/relay/tank support, and Home Assistant integration through ESPHome.

https://github.com/andrewcfitz/esphome-onecontrol

The upstream project intentionally does not publish motor controls because awnings/slides require additional deadman and safety considerations. The custom awning work in this repository builds on the protocol behavior while adding explicit timing, stop logic, motor-load endpoint detection and position tracking for the two awnings tested on this coach.

### Manos — `OneControl-RV-C-Protocol`

Manos' OneControl RV-C protocol research and implementation is another major reference for understanding Lippert OneControl device discovery, command behavior, sensor reading, and the distinction between stable function identity and runtime addressing.

https://github.com/manos/OneControl-RV-C-Protocol

Please visit and credit the upstream projects when reusing their work. This repository is intended to extend the community knowledge rather than pretend the protocol sprang fully formed from our forehead one Saturday morning.

## Development status

This is active development against a real Brinkley coach. Lights, tank sensors, water pump, tank heater and the two awnings are being validated incrementally. More OneControl devices will be added as they are positively identified and tested.

Contributions and coach-specific discoveries are welcome, but safety-related motor controls should include evidence of how the device was identified and how stop/failure behavior was validated.
