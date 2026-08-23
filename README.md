# ESP32 LED Strip Controller

*[Version française](README.fr.md)*

## Overview

This project replaces the proprietary remote-control box of a 12V RGB LED
strip with an ESP32 running [ESPHome](https://esphome.io/), integrated into
Home Assistant. It's a learning project first: every step below favors
understanding *why* something works over just getting it to work, and this
document is meant to grow alongside the build rather than being written
after the fact.

**Status:** hardware and tooling inventoried, ESPHome config adapted for
the board on hand, all 3 MOSFETs individually bench-tested and validated.
Full 3-channel RGB circuit wired end-to-end on breadboard, flashed, and
validated stable — individual channels plus full white at 100% brightness.

**Planned order of operations:**
- [x] Strip measurements (common-anode confirmed, see
      [Strip measurements & findings](#strip-measurements--findings))
- [x] Hardware & tooling inventory
- [x] Adapt `bandeau-salon.yaml` to the ESP32-C6 (board id, framework,
      GPIO pins) — config validated, not yet flashed
- [x] Bench-test each of the 3 MOSFETs individually (multimeter continuity,
      independent of the ESP32) — see
      [`mosfet-test-schema.html`](mosfet-test-schema.html)
- [x] Wire and validate a single channel end-to-end on breadboard (GPIO →
      MOSFET → strip wire), flashing a minimal one-output config — see
      [`cablage-canal-rouge.html`](cablage-canal-rouge.html) and
      [Flashing & first run](#flashing--first-run)
- [x] Replicate to the other 2 channels — see
      [`cablage-canaux-vert-bleu.html`](cablage-canaux-vert-bleu.html)
- [x] Full 3-channel ESPHome config, flash, per-channel test — see
      [Flashing & first run](#flashing--first-run)
- [ ] Home Assistant integration
- [ ] Final assembly (perfboard, buck converter setup, enclosure) — see
      [Final assembly](#final-assembly)

Note: the ESP32 stays on USB power throughout breadboard prototyping — the
buck converter is only needed for the final assembly step, so its setup is
deferred until then. Once the strip's real 12V supply is connected (from
the single-channel wiring step onward), the ESP32's USB ground and the
strip supply's ground must share a common rail on the breadboard.

**Key decisions so far:**
- WiFi + ESPHome rather than native Zigbee, for simplicity and the depth of
  available documentation. A native Zigbee version (ESP32-C6/H2) is a
  possible future iteration, see [Future ideas](#future-ideas).
- The MOSFET switching circuit is hand-wired from bare components (not a
  pre-built module), specifically to understand and build the circuit from
  scratch rather than treat it as a black box.
- The ESPHome config is managed locally via the `esphome` CLI and versioned
  in this repository, rather than through the Home Assistant add-on.

## Table of contents

1. [Hardware](#hardware)
2. [Tooling](#tooling)
3. [Strip measurements & findings](#strip-measurements--findings)
4. [MOSFET switching circuit](#mosfet-switching-circuit)
5. [ESPHome configuration](#esphome-configuration)
6. [Flashing & first run](#flashing--first-run)
7. [Home Assistant integration](#home-assistant-integration)
8. [Final assembly](#final-assembly)
9. [Future ideas](#future-ideas)

## Hardware

The LED strip is a non-addressable common-anode RGB strip, originally driven
by a small proprietary remote-control box (color + brightness).

Bill of materials for the ESP32 replacement:

| Part | Notes |
|---|---|
| **ESP32-C6-DevKitC-1** (box: `ESP32-C6-DEV-KIT-N8-M`, module: `ESP32-C6-WROOM-1`) | RISC-V, 8MB flash. Not the classic WROOM-32 the original plan assumed — see note below. |
| 3x N-channel logic-level MOSFET, **IRLB8721** | Logic-level, saturates fully from 3.3V GPIO |
| 3x 220Ω resistor (from assortment kit, 10Ω–1MΩ) | Gate resistor, limits inrush current charging the gate |
| 3x 10kΩ resistor (from assortment kit, 10Ω–1MΩ) | Gate pull-down, keeps the MOSFET off by default during ESP32 boot/reset |
| 12V→5V buck converter, **MP1584EN** | To power the ESP32 from the strip's own 12V supply in the final build |
| DC jack (female, 5.5×2.1mm) → screw terminal adapter | Taps +12V/GND from the existing supply without cutting its original cable |

*(Remaining to double-check: the MP1584EN module's output must be set to a
verified 5V before connecting the ESP32 — see note below.)*

> **Note — buck converter setup:** the MP1584EN module is adjustable via a
> small trimmer potentiometer, not fixed-output. Before wiring it to the
> ESP32, set the output to 5V and verify with a multimeter under no load —
> connecting an unregulated/mis-set output could damage the board.

> **Note — board substitution:** the board on hand is an ESP32-C6, not the
> generic ESP32-WROOM-32 the plan originally assumed. The C6 is RISC-V
> rather than Xtensa; the [ESPHome configuration](#esphome-configuration)
> has been updated accordingly (`board: esp32-c6-devkitc-1`, `esp-idf`
> framework, GPIO18/19/20). Upside: the C6 has native 802.15.4
> (Zigbee/Thread) alongside WiFi, so the [native Zigbee](#future-ideas)
> path no longer requires separate hardware.

## Tooling

Tools already on hand, useful from this first breadboard-prototyping phase
onward:

| Tool | Purpose |
|---|---|
| Breadboard | Solderless prototyping of the MOSFET switching circuit |
| Dupont wires | Breadboard-to-breadboard and breadboard-to-ESP32 connections |
| Precision screwdriver kit | Adjusting the MP1584EN buck converter's trimmer, general small-part work |
| Adjustable bench power supply (**MEGO**, EIM Technology), plugs directly onto the breadboard rails | Bench-testing the MOSFET circuit at a known, controllable voltage before connecting the real 12V strip supply |
| Multimeter | Continuity/voltage checks — strip measurements, MOSFET gate/drain-source behavior, buck converter output verification |

## Strip measurements & findings

Measured 2026-08-09, with the original control box still in place:

- Supply voltage: **12V DC**.
- One wire stays at a constant 12V regardless of color/brightness settings
  → this is the common **V+** anode.
- The other three wires (R/G/B) drop to ~0.3–0.5V at full brightness/full
  white, and rise back up (towards ~10V) as a channel's brightness is
  lowered → each channel is switched **on the ground side** via PWM.
- Isolating a single color (e.g. red only) drops only that wire while the
  other two rise back up → confirms the wire-to-channel mapping and that
  the three channels are independent.

**Conclusion:** common-anode RGB strip, ground-side PWM switching per
channel. This is exactly what the MOSFET circuit below needs to reproduce.

## MOSFET switching circuit

*(Not wired yet — this section will be filled in during breadboard
prototyping.)*

Planned circuit, repeated identically for each of the 3 channels (R, G, B):

```
ESP32 GPIO --[R 220Ω]--> Gate ----+
                                   |
                          [R 10kΩ to GND]  (pull-down)
                                   |
                        Drain -- strip's R/G/B wire
                        Source -- common ground
```

- **Gate**: driven from the GPIO through the ~220Ω resistor. The 10kΩ
  pull-down keeps the MOSFET off whenever the GPIO isn't actively driving
  3.3V — without it, the floating GPIO state during ESP32 boot/reset could
  randomly turn a channel on.
- **Drain**: wired directly to the corresponding R/G/B wire of the strip.
- **Source**: wired to common ground.
- The strip's **V+** connects directly to the 12V supply (bypassing the
  original control box entirely).
- **Common ground is mandatory** between the ESP32, the 3 MOSFETs and the
  strip's power supply — without it, "3.3V" on a GPIO has no defined
  meaning from the MOSFET's perspective.

Before wiring the strip in: check each MOSFET individually with a
multimeter (continuity mode), confirming Gate voltage toggles Drain-Source
continuity on/off as expected.

## ESPHome configuration

See [`bandeau-salon.yaml`](bandeau-salon.yaml).

- `esp32: board: esp32-c6-devkitc-1`, `framework: type: esp-idf` — matches
  the ESP32-C6-DevKitC-1 actually on hand (RISC-V; the `esp-idf` framework
  has more mature ESP32-C6 support in ESPHome than `arduino`).
- `output: platform: ledc` — one PWM output per color channel, on
  **GPIO18/19/20**. Picked from the ESP32-C6-DevKitC-1's conflict-free GPIO
  pool (0,1,2,3,6,7,10,11,18,19,20,21,22,23), avoiding: strapping pins
  GPIO4/5/8/9/15, native-USB GPIO12/13, and the UART console GPIO16/17.
- `light: platform: rgb` — combines the three outputs into one light entity.
- Secrets (WiFi credentials, OTA password, API encryption key) live in
  `secrets.yaml`, gitignored; `secrets.yaml.example` is the committed
  template.

> **Note — locating the GND pins:** the ESP32-C6-DevKitC-1 has 4 GND pins
> (all internally tied together, any of them works), spread across the two
> J1/J3 headers (pinout confirmed via the official Espressif docs):
> - J1, position 15 (last, right after 5V)
> - J3, position 1 (first, right before TX/RX)
> - J3, position 12 (between GPIO9 and GPIO13)
> - J3, position 15 (last, after GPIO12)
>
> Handy: GPIO18/19/20 sit on J3, positions 10/9/8 — right next to the GND
> at J3 position 12 (2 slots from GPIO18). The 3 PWM outputs and one GND
> fit within 5 consecutive positions on the same header. Positions
> physically confirmed on the board on hand on 2026-08-20.

Validated with `esphome config bandeau-salon.yaml` (config parses and
resolves cleanly — this does not yet compile or flash).

## Flashing & first run

Red channel validated 2026-08-21 with
[`bandeau-salon-test-rouge.yaml`](bandeau-salon-test-rouge.yaml) — a
temporary single-output variant of `bandeau-salon.yaml` (`light: platform:
monochromatic` on `output_red` only), see
[`cablage-canal-rouge.html`](cablage-canal-rouge.html) for the wiring.

- First flash over USB: `esphome run bandeau-salon-test-rouge.yaml`. On the
  ESP32-C6-DevKitC-1, the board enumerates as an Espressif "USB JTAG/serial
  debug unit" (`idVendor=303a`) — pick that serial device if several are
  offered. The USB cable must carry data lines, not just power (a
  charge-only phone cable will make the board's power LED light up but
  never enumerate on the host).
- The light entity was controlled from Home Assistant (auto-discovered via
  the ESPHome integration/mDNS once flashed) rather than from the ESPHome
  CLI, which has no way to toggle an entity directly.
- Checked the logs for a successful WiFi connection, then ramped brightness
  0% → 100%.
- **Finding:** at ~99-100% brightness (near-continuous current draw,
  vs. the lower average current of a partial PWM duty cycle), the strip cut
  out — traced to the MEGO bench supply's current limiting, not a wiring or
  MOSFET fault (confirmed by ESPHome logs showing no ESP32 reboot/WiFi drop
  at the same moment). Slightly lowering the MEGO's set voltage brought the
  draw back under the limit; stable at 100% since. Not expected to recur on
  the strip's real 12V supply, which is sized for full 3-channel load.
- MOSFET stayed barely warm after several minutes at 100% — validated.
- Subsequent flashes can be done over-the-air (OTA) once the device is on
  the network.

Full 3-channel RGB validated 2026-08-23 with
[`bandeau-salon.yaml`](bandeau-salon.yaml), wiring per
[`cablage-canaux-vert-bleu.html`](cablage-canaux-vert-bleu.html):

- Flashed over USB (new device identity vs.
  `bandeau-salon-test-rouge`, so USB was required again for this first
  flash of `bandeau-salon`).
- **Finding:** driving the blue channel lit up green instead — traced to
  the green/blue gate wires being swapped between GPIO19 and GPIO20 on the
  breadboard, not a MOSFET or config issue. Fixed by correcting the gate
  wiring; re-tested red/green/blue individually afterwards to confirm the
  swap was fully resolved in both directions.
- Full white (all 3 channels at 100% simultaneously) hit the MEGO's
  current-limit trip again, this time at its original set voltage — the
  combined 3-channel draw is higher than any single channel. Lowering the
  MEGO to **9.5V** brought it back under the limit; stable at full white
  since. Same bench-supply-only caveat as the Red-channel finding above:
  not expected on the strip's real 12V supply, which is sized for the full
  load.
- All 3 MOSFETs stayed cool after sustained full-white load — validated.

## Home Assistant integration

*(Not done yet.)*

- The native **ESPHome** integration should auto-discover the device via
  mDNS (Settings → Devices & Services).
- Verify color/brightness control from the dashboard and check for
  flicker (may require adjusting the `ledc` PWM frequency).

## Final assembly

*(Future — after the breadboard prototype is validated.)*

- Re-solder the MOSFET circuit onto perfboard.
- Heat-shrink the 12V/GND joints.
- Removable screw-terminal connectors for the external wiring.
- Plastic enclosure (not metal — would shield the ESP32's PCB antenna).
- Add a fuse/polyfuse on +12V and a decoupling capacitor near the ESP32
  supply.

## Future ideas

- **Native Zigbee**: ESP32-C6 (WiFi + BLE + 802.15.4) or ESP32-H2, joining
  the existing Zigbee mesh via the Nabu Casa ZBT-2 coordinator, using
  Espressif's official `HA_color_dimmable_light` example (ESP-IDF, outside
  ESPHome). Only worth exploring once the WiFi version is validated and
  working.
