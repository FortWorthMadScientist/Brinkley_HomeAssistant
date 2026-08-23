# ESP32 / CAN Diagnostics

This diagnostic layer exists to capture evidence when the Brinkley ESP32 controller fails intermittently. It is intentionally observational: it does not alter awning timing, OneControl commands, motor safety logic, or device mappings.

## Enable it

Place `diagnostics.yaml` in the same ESPHome directory as the main Brinkley YAML, then add this entry under the existing `packages:` block:

```yaml
  diagnostics: !include diagnostics.yaml
```

The package assumes Andrew Fitzpatrick's OneControl core has created the CAN component with the ID `can_bus`.

## What Home Assistant will show

### ESP / network health

- `ESP32 Uptime`
- `ESP32 Boot Count`
- `ESP32 Last Reset Reason`
- `ESP32 API Connected`
- `ESP32 WiFi RSSI`
- `ESP32 IP Address`
- `ESP32 Connected SSID`
- `ESP32 Connected BSSID`
- `ESP32 WiFi MAC`
- `ESP32 Device Info`
- `ESP32 Restart`

### Runtime / memory health

- `ESP32 Heap Free`
- `ESP32 Heap Minimum Free`
- `ESP32 Heap Largest Block`
- `ESP32 Heap Fragmentation`
- `ESP32 Max Loop Time`
- `ESP32 CPU Frequency`

### CAN health

- `CAN RX Frame Count`
- `CAN RX Standard Frame Count`
- `CAN RX Extended Frame Count`
- `CAN RX Rate`
- `CAN Seconds Since Last RX`

## How to interpret a failure

### Everything goes unavailable, then uptime restarts near zero

The ESP32 rebooted. Check `ESP32 Last Reset Reason` immediately after it comes back. A brownout, watchdog, panic, software reset, or other reset reason materially changes the investigation.

`ESP32 Boot Count` should also increment. This is persistent across reboots.

### Everything goes unavailable, but uptime did not reset

This points more strongly toward Wi-Fi, ESPHome native API connectivity, or Home Assistant/network reachability rather than a full ESP32 reboot.

Review the Home Assistant history for `ESP32 WiFi RSSI` and `ESP32 API Connected` immediately before the outage.

### ESP entities remain available, but CAN counters stop increasing

The ESP32 and API are alive but CAN receive traffic has stopped. Investigate the CAN transceiver, wiring, bus state, or CAN component.

`CAN Seconds Since Last RX` will begin climbing while `CAN RX Frame Count` remains unchanged.

### CAN rate becomes abnormal before a failure

Use `CAN RX Rate` to look for an unusual traffic flood or abrupt traffic collapse before the failure. Compare it with heap and loop timing.

### Heap shrinks or loop time spikes before a reboot

`ESP32 Heap Minimum Free`, `ESP32 Heap Fragmentation`, `ESP32 Heap Largest Block`, and `ESP32 Max Loop Time` can expose memory pressure or a blocked main loop. A severely blocked loop can contribute to watchdog resets or network disconnects.

## Home Assistant history recommendation

Keep history enabled for these entities while diagnosing intermittent failures. The most useful evidence is often the 1–5 minutes before the controller disappears.

At minimum retain history for:

- ESP32 Uptime
- ESP32 Boot Count
- ESP32 Last Reset Reason
- ESP32 WiFi RSSI
- ESP32 Heap Minimum Free
- ESP32 Max Loop Time
- CAN RX Rate
- CAN Seconds Since Last RX

## Why the CAN observer is low risk

The diagnostic CAN handlers only increment counters and record `millis()` when frames are received. They do not transmit CAN frames, change node addresses, alter remote-control sessions, or touch awning motor logic.

If diagnostic overhead itself ever becomes suspect, remove the single `diagnostics: !include diagnostics.yaml` package line and recompile to return to the prior controller behavior.
