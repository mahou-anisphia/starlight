---
title: MikroTik hAP ax² — Non-DFS 5GHz Channel Pinning
description: Pinning the 5GHz radio to a private non-DFS channel pool to stop radar-triggered outages, plus the country setting and a known firmware bug.
---

## What this is

The 5GHz WiFi configuration on the MikroTik hAP ax² (RouterOS 7.23.1,
`wifi-qcom` driver) serving SSID `MagicologyLab 5GHz` on interface `wifi1`.
The radio is locked to a dedicated channel profile (`mine-5g`) that only lists
non-DFS frequencies, so it can never select a radar-shared channel.

## Why

The 5GHz radio kept going silent while the router stayed powered and
LAN-reachable. Cause was a **DFS radar event**: the radio had roamed onto a
DFS channel (5700 MHz), RouterOS detected a radar pattern, and regulatory rules
forced it to vacate — a ~30-minute non-occupancy lockout plus a channel
availability check before it could transmit again. In a dense concrete building
this fires often (real radar, reflections, or false positives), so the fix is
to remove DFS channels from the equation entirely rather than tolerate repeated
outages.

Non-DFS 5GHz channels on this radio: UNII-1 (5180, 5200, 5220, 5240) and
UNII-3 (5745, 5765, 5785, 5805, 5825). Everything from 5260–5720 is DFS and is
deliberately excluded. UNII-3 was chosen (top of band, typically quieter,
higher allowed tx-power).

## Configuration

Country — must be the exact RouterOS string `Viet Nam` (two words, quoted).
Set on both radios:

```
/interface/wifi/set wifi1 configuration.country="Viet Nam"
/interface/wifi/set wifi2 configuration.country="Viet Nam"
```

Non-DFS-only channel profile, pinned to wifi1:

```
/interface/wifi/channel/add \
  name=mine-5g \
  frequency=5745,5765,5785,5805 \
  width=20/40/80mhz

/interface/wifi/set wifi1 channel=mine-5g
```

Resulting `wifi1` state:

| Setting           | Value                             |
| ----------------- | --------------------------------- |
| SSID              | `MagicologyLab 5GHz`              |
| Mode              | `ap`                              |
| Country           | `Viet Nam`                        |
| Channel profile   | `mine-5g`                         |
| Frequencies       | `5745,5765,5785,5805`             |
| Band              | `5ghz-ax`                         |
| Width             | `20/40/80mhz`                     |
| skip-dfs-channels | `10min-cac` (secondary safeguard) |
| Security          | `wpa2-psk,wpa3-psk`               |

Verify the radio is on air and off DFS:

```
/interface/wifi/monitor wifi1 once
```

Expect `state: running`, a `channel` in the 5745–5805 range, and **no `/D`
flag** anywhere.

## Gotchas / notes

- **"Viet Nam" is the only accepted spelling.** `Vietnam` and `etsi` both fail
  with `input does not match any value`. The value has a space, so it must be
  quoted. Use Tab-completion on `configuration.country=` to see the canonical
  list if it ever changes between builds.

- **The `mine-5g` profile lists only non-DFS frequencies by design.** Listing
  four (not one fixed channel) lets RouterOS pick the cleanest at startup and
  hop within the private pool, while making a radar channel structurally
  impossible. To nail it to a single channel instead, use `frequency=5745`
  alone.

- **Known firmware bug — "failed to set country" / inactive radio.** On
  RouterOS 7.2x with `wifi-qcom`, the radio can come up with flag `I`
  (inactive) and comment `failed to set country`, even with a valid country
  string. This is a recurring regression, not a config error. **Fix: reboot**
  (`/system/reboot`) — it clears the flag and the config survives. If it
  recurs often, a nightly scheduled reboot is the community workaround; a full
  netinstall of matching `routeros` + `wifi-qcom` packages has fixed it
  permanently in older cases. Do not chase different country names — none will
  activate while the radio is in this state.

- **`B` vs `R` flag:** after a reboot wifi1 shows `M B` (bound) rather than
  `M BR` (running) until a 5GHz client associates. Bound = enabled and
  broadcasting; it is not disabled. Disabled would show an `X` flag.

- **Width in a concrete building:** `20/40/80mhz` negotiates down gracefully,
  but 40MHz is the realistic stable ceiling through walls — 80MHz needs four
  clean adjacent channels that usually aren't available. Drop to `20/40mhz` if
  prioritising stability over peak throughput.

- **Unrelated open issue:** the logs showed 5GHz clients reconnecting every
  ~5 minutes for days, predating the DFS event. That is a separate
  roaming/band-steering or client problem and is _not_ addressed by this
  channel pinning.
