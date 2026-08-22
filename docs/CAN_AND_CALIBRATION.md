# CAN Bus and Awning Calibration Guide

This document is the commissioning checklist for the ESP32 OneControl bridge.

## What is connected to what?

The ESP32 does not connect directly to CAN-H/CAN-L. Its CAN controller TX/RX pins connect to a CAN transceiver. The transceiver provides the differential CAN-H and CAN-L connection to the RV.

Current development pin assignment:

| Signal | ESP32 |
|---|---|
| CAN controller TX | GPIO5 |
| CAN controller RX | GPIO4 |

The OneControl network on the development coach is 250 kbit/s.

## Before transmitting

First prove the ESP32 is receiving CAN traffic. ESPHome logs should show changing OneControl status frames while physical controls are operated. If nothing is received, check power, CAN-H/CAN-L, transceiver wiring, bitrate and termination before changing protocol code.

Do not transmit to an address until the device has been positively identified on that coach.

## Finding the calibration values

Open `esphome/brinkley-can.yaml` and search for:

`USER CALIBRATION - AWNING TIMERS / SAFETY`

The values are intentionally together so an owner does not need to edit the motor-control implementation.

```yaml
awning_1_full_extend_ms: "21500"
awning_2_full_extend_ms: "22500"
awning_1_retract_stall_threshold: "3000"
awning_2_retract_stall_threshold: "3300"
```

## Calibrating extension time

1. Fully retract the awning.
2. Start with an extension value shorter than expected.
3. Command Extend while physically watching the awning.
4. Measure how far short it stops.
5. Increase the value in small increments.
6. Repeat until the desired full-extension point is reached without overrun.

The number is milliseconds. 21.5 seconds is `21500`.

Do not deliberately overrun an awning to find the number. On some awnings the roller can pass the desired point and begin wrapping the fabric in the opposite direction.

## Calibrating retract stall

Motor-load telemetry rises when the awning reaches the fully closed mechanical stop.

1. Observe normal running load while retracting.
2. Observe the load when the awning is physically fully closed.
3. Select a threshold high enough that normal travel does not trigger it, but below the repeatable closed-end load.
4. Test several complete retract cycles.
5. Verify it closes snugly without holding the motor against the stop longer than necessary.

The implementation requires repeated high-load observations rather than trusting one spike.

## Position tracking

The position sensor is dead reckoning based on motor run time. It is useful, but it is not an absolute encoder.

The estimate is updated by both Home Assistant commands and observed factory-panel motor commands. A confirmed full-retract stall resets the estimate to 0%, providing a physical reference point.

If the estimate ever becomes questionable, fully retracting the awning is the preferred recalibration action.

## Why factory-panel tracking matters

Home Assistant may not be the only controller. Someone can extend an awning from the original OneControl panel and then leave the RV. Home Assistant still needs enough state knowledge to safely retract it later.

This is also a prerequisite for planned wind protection: an anemometer can eventually trigger automatic retraction above a configured wind threshold, regardless of whether the awning was originally extended from Home Assistant or the factory panel.

## Termination reminder

CAN normally wants exactly two 120-ohm terminators. If the ESP32/transceiver is a mid-bus tap, do not add a third terminator. If the installation replaces a terminator or becomes a physical endpoint, provide the required termination. Verify the actual topology rather than guessing.
