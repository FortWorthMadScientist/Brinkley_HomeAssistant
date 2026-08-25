# ESP32 + ESPHome + OneControl CAN — Complete Installation and Verification Guide

> **Start here if you are building the ESP32 controller.** This guide covers the complete path from a bare ESP32 through ESPHome installation, YAML configuration, first flash, Home Assistant integration, CAN wiring, logs, verification, commissioning, diagnostics, and awning calibration.

This guide assumes moderate technical skill and a Home Assistant installation with ESPHome Device Builder. The Brinkley/OneControl CAN network observed on the development coach operates at **250 kbit/s**.

> **Safety:** OneControl node addresses can be coach-specific. Do not blindly copy addresses from this repository to another RV. Verify devices on your own coach before enabling control, especially motorized equipment.

## 1. What you need

- Home Assistant running and accessible
- ESPHome Device Builder installed in Home Assistant
- ESP32 development board
- CAN transceiver compatible with the ESP32
- USB **data** cable
- Computer with a modern browser
- Wi-Fi credentials for the network the ESP32 will use
- CAN-H and CAN-L access to the RV OneControl CAN bus
- The project YAML file: `esphome/brinkley-can.yaml`

A typical installation has two distinct interfaces:

```text
ESP32                         CAN Transceiver

GPIO TX  -------------------> TXD
GPIO RX  <------------------- RXD
3.3V/5V --------------------> VCC (as required by your transceiver)
GND ------------------------> GND

                               CAN-H ----> RV CAN-H
                               CAN-L ----> RV CAN-L
```

**CAN-H and CAN-L are not RX and TX wires.** CAN is differential; both wires participate in transmit and receive.

The current development configuration uses:

```yaml
can_tx_pin: GPIO5
can_rx_pin: GPIO4
```

Change these values if your hardware uses different pins.

## 2. Install ESPHome in Home Assistant

In Home Assistant:

1. Open **Settings → Add-ons → Add-on Store**.
2. Search for **ESPHome Device Builder**.
3. Install it.
4. Enable **Start on boot**.
5. Enable **Watchdog**.
6. Enable **Show in sidebar** if desired.
7. Start the add-on.

ESPHome Device Builder should now appear in Home Assistant.

## 3. Connect the ESP32 to your computer

For the first installation, connect the ESP32 to your computer over USB.

Use a **data-capable USB cable**. Some cables provide power only, which is an excellent way to spend twenty minutes troubleshooting a cable pretending to be useful.

Do not connect the prototype to the RV CAN bus yet if you are still bench-testing the board.

## 4. Create the ESPHome device

Open **ESPHome Device Builder** and select **New Device**.

Suggested values for this project:

```text
Device name: brinkley-can
Friendly name: Brinkley CAN
```

For a common ESP32 development board, the configuration is typically:

```yaml
esp32:
  board: esp32dev
  framework:
    type: esp-idf
```

ESPHome may generate a basic starter YAML. That is fine; the project YAML will replace it.

## 5. Copy the project YAML

From this repository, open:

`esphome/brinkley-can.yaml`

In ESPHome Device Builder:

1. Open the `brinkley-can` device.
2. Select **Edit**.
3. Replace the generated YAML with the complete project YAML.
4. Save it.

Replacing the complete file is safer than manually merging sections. YAML is indentation-sensitive, and one misplaced space can turn an RV integration project into a punctuation investigation.

## 6. Configure Wi-Fi safely

Do not commit real Wi-Fi passwords to GitHub.

Create or edit:

`/config/esphome/secrets.yaml`

Example:

```yaml
wifi_ssid: "YOUR_WIFI_NAME"
wifi_password: "YOUR_WIFI_PASSWORD"
```

Reference those secrets from the main YAML:

```yaml
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none
```

For an always-on controller, disabling ESP32 Wi-Fi power saving can provide a steadier ESPHome API connection.

## 7. Configure CAN pins

Near the top of the project YAML is a substitutions section containing the ESP32-side CAN pins.

Example:

```yaml
substitutions:
  can_tx_pin: GPIO5
  can_rx_pin: GPIO4
```

Wire the logic side of the transceiver as follows for this configuration:

```text
ESP32 GPIO5  ---> CAN transceiver TXD
ESP32 GPIO4  <--- CAN transceiver RXD
ESP32 GND    ---> CAN transceiver GND
ESP32 power  ---> CAN transceiver VCC, at the voltage required by the module
```

The CAN side of the transceiver connects to the RV:

```text
CAN transceiver CAN-H ---> RV CAN-H
CAN transceiver CAN-L ---> RV CAN-L
```

**Never connect ESP32 GPIO pins directly to CAN-H or CAN-L.** The CAN transceiver is required.

## 8. Understand CAN termination

A normal CAN network usually has two 120-ohm terminating resistors, one at each physical end:

```text
120Ω ---------------- CAN BUS ---------------- 120Ω
```

Do not add a third terminator simply because your CAN module includes one.

If the ESP32 is a mid-bus tap and the original network remains properly terminated, the ESP32 transceiver normally should not add another termination resistor. If your installation replaces an endpoint, the situation may be different. Verify the topology rather than guessing.

## 9. Validate the YAML before installing

Inside ESPHome Device Builder, run **Validate**.

You want the configuration to complete without errors. Do not install firmware until validation succeeds.

Pay particular attention to errors involving:

- invalid options
- missing IDs
- unknown components
- YAML indentation
- lambdas
- substitutions

Warnings are not always fatal. Errors are.

## 10. Perform the first USB installation

For a new ESP32:

1. Select **Install**.
2. Choose the USB/browser installation method available in ESPHome.
3. Select the ESP32 serial device when prompted.
4. Allow ESPHome to compile and flash the firmware.

Common USB serial names include CP210x, CH340, USB Serial, or UART Bridge.

The first compile can take several minutes. After flashing, the ESP32 will reboot.

## 11. Verify Wi-Fi connectivity

After boot, ESPHome should eventually show the device as online.

Home Assistant normally discovers ESPHome devices automatically. If it does not, open **Settings → Devices & Services → ESPHome** and add the ESP32 using its IP address.

Useful signs in the logs include successful Wi-Fi connection, an assigned IP address, and a successful ESPHome API connection.

## 12. Verify the ESPHome API and controller diagnostics

Open the Brinkley CAN device in Home Assistant.

The diagnostic build exposes information such as:

- ESP32 API Connected
- ESP32 Uptime
- ESP32 Boot Count
- ESP32 Last Reset Reason
- ESP32 WiFi RSSI
- ESP32 Heap Free
- ESP32 Heap Minimum Free
- ESP32 Heap Largest Block
- ESP32 Heap Fragmentation
- ESP32 Max Loop Time
- ESP32 IP Address

These diagnostics help distinguish an ESP32 reboot from a Wi-Fi/API problem or a CAN-side failure.

## 13. Power down before connecting to the RV CAN bus

Once firmware, Wi-Fi, and the API have been verified:

1. Power the ESP32 off.
2. Connect CAN-H and CAN-L to the verified OneControl CAN connection.
3. Double-check polarity and termination.
4. Power the ESP32 back on.

Reversing CAN-H and CAN-L normally prevents communication even though the ESP32 itself may remain perfectly healthy on Wi-Fi.

## 14. Open ESPHome logs

In ESPHome Device Builder, open **Logs** for the Brinkley CAN device.

The logs should show the ESPHome connection and diagnostic sensor updates.

For CAN verification, the most useful Home Assistant entities are:

- CAN RX Frame Count
- CAN RX Rate
- CAN Seconds Since Last RX

## 15. Verify CAN reception

On an active OneControl network, the frame counter should increase continuously and the receive rate should be greater than zero.

A healthy example might look like:

```text
CAN RX Frame Count:       12456
CAN RX Rate:              103.7 frames/s
CAN Seconds Since Last RX: 0.0 s
```

The exact frame rate is not important. The development coach has previously shown roughly 100 frames per second while active.

A failure condition looks like:

```text
CAN RX Frame Count:       0
CAN RX Rate:              0.0 frames/s
CAN Seconds Since Last RX: Unknown
```

That means the ESP32 has not recorded a CAN frame since boot. Wi-Fi/API connectivity does **not** prove that CAN is working.

## 16. Verify passive CAN sensors first

Before transmitting commands, verify passive data.

Currently mapped tank sensors on the development coach include:

- Fresh Tank
- Grey Tank 1
- Grey Tank 2
- Grey Tank 3
- Black Tank

The exact percentages depend on the coach. What matters during commissioning is that they stop showing `Unknown`/`Unavailable` and begin reporting actual values.

## 17. Verify mapped light state

Currently mapped lights on the development coach are:

```text
Kitchen Pendants  0xCB
Island Light      0xC9
Awning Light 1    0x3A
Awning Light 2    0x88
```

These addresses are development evidence, **not guaranteed universal Brinkley addresses**. Discover and verify node addresses on another coach before enabling control.

## 18. Check light initialization

The upstream OneControl package intentionally delays normal light commands after boot so real CAN state has time to populate.

The diagnostic configuration exposes:

```text
DEBUG Light Ready
DEBUG Light Sync
```

After approximately 10–15 seconds, the expected state is:

```text
DEBUG Light Ready = ON
DEBUG Light Sync  = OFF
```

If `DEBUG Light Ready` remains OFF, normal light commands will be blocked.

During development, a custom `on_boot` configuration interfered with this upstream initialization. The project configuration was corrected so local boot diagnostics do not replace the upstream light-ready initialization.

## 19. Test a low-risk device first

Do not make the first transmitted CAN command an awning, slide, pump, or heater.

Use a known light.

For example, test Kitchen Pendants:

1. Turn Kitchen Pendants ON from Home Assistant.
2. Confirm the physical light turns on.
3. Confirm CAN feedback reports ON.
4. Turn Kitchen Pendants OFF.
5. Confirm the physical light turns off and the state returns to OFF.

A light is an excellent commissioning target because a failed command is inconvenient rather than mechanically exciting.

## 20. Understand command and state feedback

Home Assistant does not need to blindly assume that a command worked. The expected path is:

```text
Home Assistant
      |
      | command
      v
ESP32 / ESPHome
      |
      | OneControl CAN command
      v
OneControl node
      |
      | CAN status feedback
      v
ESP32 / ESPHome
      |
      v
Home Assistant state
```

If Home Assistant briefly changes state and then returns to the previous state, the command may have been attempted but the physical node continued reporting the old state.

## 21. Water pump and tank heater

The development configuration includes:

- Water Pump
- Tank Heater

These are latching relay controls and use the upstream OneControl relay behavior.

Do not test them remotely during initial commissioning. Test them while physically at the RV so you can immediately correct an unexpected state.

## 22. Awning controls

The custom controller currently provides:

- Awning 1 Extend
- Awning 1 Retract
- Awning 1 Stop
- Awning 2 Extend
- Awning 2 Retract
- Awning 2 Stop

It also exposes diagnostics including:

- motor load
- movement state
- estimated position
- estimated remaining extension time

The position is estimated from motor run time. It is not an absolute encoder reading.

## 23. Calibrate awning extension and retraction

The main YAML intentionally groups owner-adjustable values in a clearly marked section:

`USER CALIBRATION - AWNING TIMERS / SAFETY`

Development-coach examples:

```yaml
awning_1_full_extend_ms: "21500"
awning_2_full_extend_ms: "22500"
awning_1_retract_stall_threshold: "3000"
awning_2_retract_stall_threshold: "3300"
```

### Extension time

`21500` means 21.5 seconds.

If an awning extends too far, decrease the value. For example:

```yaml
awning_1_full_extend_ms: "20500"
```

means 20.5 seconds.

Calibrate conservatively while physically watching the awning. Some awnings can continue past the desired extension point and begin rolling the fabric backward.

### Retract stall threshold

The controller watches motor load during retraction. Repeated readings above the configured threshold are used to identify the fully retracted mechanical endpoint.

Set this using **your own motor-load telemetry**. Too low can stop the awning short; too high can hold the motor against the stop unnecessarily.

## 24. Awning position tracking

The controller accumulates actual motor run time rather than restarting a complete extension timer every time Extend is pressed.

Example:

```text
Full extension travel: 20 seconds
Already extended:       5 seconds
Estimated remaining:   15 seconds
```

This prevents a partially extended awning from receiving another complete full-extension run and wrapping backward at the end.

Retraction subtracts elapsed movement from the estimate. A confirmed fully retracted stall resets the position estimate to 0%, providing a physical calibration reference.

## 25. Factory-panel position tracking

The ESP32 also watches relevant factory-panel CAN motor commands.

Conceptually:

```text
Factory panel moves awning
          |
          v
ESP32 observes CAN command
          |
          v
Estimated position updates
```

This matters because an awning may be moved from the factory panel and later controlled from Home Assistant. Home Assistant should not assume the awning is retracted simply because Home Assistant did not extend it.

It also lays the groundwork for future safety automation such as automatic retraction when a validated wind sensor reports excessive wind.

## 26. OTA updates

After the first USB installation, later firmware updates can normally be installed wirelessly.

Typical workflow:

1. Edit YAML.
2. Validate.
3. Select **Install**.
4. Choose the wireless/OTA option.
5. Wait for compilation, upload, and reboot.

If OTA fails, first verify the ESP32 IP address, Wi-Fi RSSI, and API connectivity before changing firmware logic.

## 27. Logging levels

ESPHome logging can range from ERROR/WARN through INFO/DEBUG/VERBOSE.

CAN networks can produce large amounts of traffic. Logging every CAN frame can unnecessarily consume ESP32 processing, API bandwidth, and Wi-Fi capacity.

A stable deployment can use quieter logging such as:

```yaml
logger:
  level: WARN
```

Increase logging temporarily when diagnosing a specific problem.

## 28. Useful ESP32 health numbers

### Wi-Fi RSSI

Very rough guidance:

```text
-30 dBm  Excellent
-50 dBm  Very good
-60 dBm  Good
-70 dBm  Marginal
-80 dBm  Poor
```

### Heap

Watch:

- ESP32 Heap Free
- ESP32 Heap Minimum Free
- ESP32 Heap Largest Block
- ESP32 Heap Fragmentation

A steadily falling minimum heap or severe fragmentation can point toward a memory problem.

### Loop time

Watch `ESP32 Max Loop Time`.

The current development configuration has shown loop times around 18–20 ms during healthy operation. Sudden values in the hundreds or thousands of milliseconds deserve investigation.

## 29. If every entity goes grey in Home Assistant

If all ESPHome entities become unavailable at the same time, do not immediately blame CAN.

Check:

- ESP32 API Connected
- ESP32 Uptime
- ESP32 Boot Count
- ESP32 Last Reset Reason
- ESP32 WiFi RSSI

Interpretation:

```text
Uptime reset / boot count increased
    -> ESP32 rebooted

Uptime continues but API disappears
    -> Wi-Fi/API connectivity problem

ESP/API stay alive but CAN counters stop
    -> CAN controller/transceiver/bus-side problem
```

These are different failure modes and should be diagnosed separately.

## 30. Troubleshooting CAN RX = 0

If CAN RX Frame Count remains zero, check in this order:

1. Is the OneControl network awake/active?
2. Is the ESP32 powered?
3. Is the CAN transceiver powered correctly?
4. Is transceiver ground connected to ESP32 ground?
5. Is ESP32 TX connected to transceiver TXD?
6. Is ESP32 RX connected to transceiver RXD?
7. Is CAN-H connected correctly?
8. Is CAN-L connected correctly?
9. Are CAN-H and CAN-L reversed?
10. Is termination appropriate for the tap location?
11. Is the CAN bitrate configured for 250 kbit/s?
12. Are the configured GPIO pins correct?

Do not confuse ESPHome log messages with CAN messages. ESPHome can be perfectly healthy while the CAN receive count remains zero.

## 31. Receives CAN but will not control devices

If tank/status data works but commands do not, determine whether **all** transmitted commands fail or only one device class.

If awnings work but lights do not, CAN transmit hardware is already strongly implicated as functional. Investigate the light/session logic rather than immediately rewiring CAN-H/CAN-L.

If a direct known-good OneControl command works but the normal Home Assistant entity does not, the CAN hardware and OneControl session are working; the entity wrapper or gating logic is the likely problem.

That exact diagnostic path was used during development to identify the `light_ready` initialization problem.

## 32. Recommended commissioning order

For a new permanent controller, use this order:

1. Bench-power the ESP32.
2. Flash ESPHome over USB.
3. Verify Wi-Fi.
4. Verify the Home Assistant/ESPHome API.
5. Verify ESP32 diagnostics.
6. Power the controller off.
7. Connect CAN-H/CAN-L.
8. Power the controller on.
9. Verify CAN RX traffic.
10. Verify passive tank/status sensors.
11. Test one known light.
12. Test the remaining verified lights.
13. Test pump/heater only while physically present.
14. Test Awning Stop before meaningful motor movement.
15. Test a short awning movement while physically observing it.
16. Calibrate awning extension time.
17. Calibrate retract stall threshold.
18. Verify factory-panel position tracking.
19. Observe ESP32 stability over time.
20. Only then treat the installation as production-ready.

Each stage proves one layer before introducing another.

## 33. The golden troubleshooting rule

If something stops working, do not change five things at once.

Work through the stack:

```text
Home Assistant
     |
ESPHome API
     |
ESP32 runtime
     |
ESP32 CAN controller
     |
CAN transceiver
     |
CAN-H / CAN-L
     |
OneControl node
```

Verify each layer in order. That turns a mysterious "nothing works" failure into a bounded troubleshooting problem instead of an afternoon spent threatening an ESP32 with a hammer.

## Related documentation

- Main project overview: `README.md`
- Main ESPHome configuration: `esphome/brinkley-can.yaml`
- Controller diagnostics: `docs/DIAGNOSTICS.md`

## Credits

The OneControl implementation used by this project builds on the work of Andrew Fitzpatrick's `esphome-onecontrol` project and Manos' `OneControl-RV-C-Protocol` research. See the repository README for detailed credits and upstream references.
