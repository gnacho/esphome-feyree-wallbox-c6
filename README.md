# Feyree EV Charger (wallbox, 7 kW) — ESPHome with XIAO ESP32-C6

[🇪🇸 Versión en castellano](README.es.md)

![Feyree EV wallbox charger (vendor product image)](docs/images/product.jpg)

**100% local** integration of the Feyree EV wallbox charger (Tuya, single-phase 32 A / 7 kW)
into Home Assistant, by replacing the stock **WBR3** WiFi module (Realtek RTL8720CF, not
flashable in circuit) with a **Seeed Studio XIAO ESP32-C6** running ESPHome. The ESP32 talks
to the charger's main MCU (GigaDevice **GD32F303**) using the **TuyaMCU** protocol
(UART, 9600 baud).

## MCU vs WiFi module — why this works

The charger actually has **two "brains"**:

| Chip | Role |
|---|---|
| **GD32F303** (main MCU) | Does *all* the real work: contactor, CP/proximity signalling, metering, display, RFID, safety. It speaks a simple serial protocol (TuyaMCU) and does **not** need any cloud. |
| **WBR3** (WiFi module) | Just a **serial-to-cloud bridge**: it forwards the MCU's datapoints to the Tuya cloud and back. It adds nothing to the charging logic. |

So replacing the WiFi module with an ESP32 running ESPHome gives you full local control
**without touching the power electronics at all**. The charger keeps working exactly as
before (display, RFID, protections); it just stops depending on the Tuya cloud.

### Why replace the WBR3 instead of flashing it

- The WBR3 (Realtek RTL8720CF) is **not among ESPHome's officially supported chips** (the
  docs list only RTL8710BN/BX for Realtek; LibreTiny does ship a WBR3 board definition, but
  RTL8720CF support is partial/experimental). OpenBeken support is experimental too.
- And it **cannot be flashed while soldered** anyway — the download-strap pin (PA00) and
  the log UART RX (PA15) are pads on the *underside* of the module. You must desolder it
  regardless.
- The community documented swapping it for a **CB3S** (Beken), which is well supported and
  has its TuyaMCU UART on the same footprint pads. Since desoldering is unavoidable either
  way, an ESP32 seemed the better option to me: mature ESPHome support, native TuyaMCU
  component and OTA updates. The price was redoing part of the wiring/YAML adaptation work
  the community had already done for the Beken swap.
- An alternative CB3S YAML is also included here if you prefer that route
  ([`feyree-cb3s.yaml`](feyree-cb3s.yaml)).

![Community CB3S swap soldered on the WBR3 footprint (photo: Elektroda community)](docs/images/cb3s_swap_community.jpg)

## Hardware

| Item | Detail |
|---|---|
| Charger | Feyree wallbox 7 kW (Tuya, Smart Life app), single-phase 32 A |
| Main MCU | GigaDevice **GD32F303** (TuyaMCU, 9600 8N1) |
| Stock WiFi module | **WBR3** (Realtek RTL8720CF) — removed |
| New module | **Seeed XIAO ESP32-C6** (`seeed_xiao_esp32c6`, **esp-idf** framework) |

![Inside the charger: power board (left) and control board (right)](docs/images/charger_opened.jpg)

### Wiring XIAO ESP32-C6 ↔ Feyree board

The TuyaMCU UART lives on **pads 15/16** of the WiFi module footprint (confirmed against the
official Tuya WBR3 datasheet application schematic: pin 16 = TXD, pin 15 = RXD). That UART is
also exposed on the 5-pin **RST/RX/TX/VDD/GND header** next to the module footprint, which is
much easier to solder to.

![Control board (PCB FLYBGU05A): WBR3 module, RST/RX/TX/VDD/GND header and A/B/GND/5V footprint](docs/images/board_overview.jpg)

![Close-up of the UART header and the WBR3 module](docs/images/header_detail.jpg)

![XIAO ESP32-C6 pinout (Seeed Studio)](docs/images/xiao_pinout.png)

| XIAO C6 | Feyree board | Function |
|---|---|---|
| D6 = **GPIO16 (TX)** | **RX** of header (pad 16 line) | TuyaMCU TX |
| D7 = **GPIO17 (RX)** | **TX** of header (pad 15 line) | TuyaMCU RX |
| **5V** | **5V** of the `A/B/GND/5V` footprint (RS485 area, unpopulated) | Power |
| GND | GND (any, all grounds are common) | Ground |

Notes:

- **Board revisions differ.** On our old board the header UART was *direct*; on the new one
  (PCB `FLYBGU05A 2025/10/28`) it is **crossed** (D6→RX, D7→TX). The silkscreen naming
  depends on the revision. If you get `Initialization failed at init_state 0` in a loop,
  swap the two wires — at 3.3 V it is harmless.
- **The TX/RX test points in the RS485 area are NOT the TuyaMCU UART** — they are another
  GD32 UART (the Modbus/RS485 one, transceiver unpopulated). Verified with continuity: they
  don't connect to pads 15/16. You will never get a handshake there.
- Power the C6 from the **5V footprint** (the XIAO regulates 3.3 V internally). Do **not**
  use the header's VDD pin (no usable current — the display flickers) and check the 3V3 rail
  with a multimeter first: ours was dead (1.2 V). Any GND works.
- **Solder all four wires.** Loose dupont jumpers caused us days of corrupt UART frames
  (~2% checksum errors) and ignored commands.

![The four wires soldered to the header pads (TX/RX crossed on this board revision)](docs/images/soldering.jpg)
- The TuyaMCU UART is the C6's hardware **UART0** (GPIO16/17). The logger goes over the
  native USB (`USB_SERIAL_JTAG`), so it does not clash with UART0.
- **Do not connect USB and board power at the same time** if you are unsure about isolation.
- If a CB3S (or the WBR3) is still sitting on the module footprint unpowered, it hangs on the
  UART lines and degrades them. Remove it.

## Datapoint (dpID) map

Scale factors confirmed by *stonacek*'s teardown on Elektroda (see References):

| dpID | Type | Description | Factor | Notes |
|---|---|---|---|---|
| 14 | enum | Work mode | — | 0=Charge now, 1=Percentage, 2=Energy, 3=Scheduled |
| 18 | bool | General charge enable | — | Master switch |
| 102 | value | Voltage L1 | ×0.1 | V |
| 105 | value | Current L1 | **×0.1** | A (raw in tenths; **not** ×0.01) |
| 109 | value | Power | ×0.1 | kW |
| 110 | value | Temperature | ×0.1 | °C |
| 112 | value | Session energy | ×0.1 | kWh, resets each session (no `state_class`) |
| 115 | value | Charge current setpoint | — | A (6–32, number) |
| 124 | enum | Charge command/state | — | 0=Charge, 1=Stop, 2=Ready, 3=Waiting |

### Starting a real charge

The dpID 18 switch alone is not enough. Confirmed sequence:

1. `dpID 14 = 0` (charge now) — clears timers/schedules.
2. `dpID 124 = 1` (Stop) — clears the previous session.
3. `dpID 124 = 0` (Charge) — starts charging.

### ⚠️ Current cannot be changed mid-charge

Per the official Feyree manual, the current (dpID 115) **can only be set before charging
starts**. Changing it during an active session triggers an "Over Current" error and stops
the charge (also reported in tuya-local). For dynamic solar charging / load balancing, the
safe pattern is:

1. `dpID 124 = 1` (Stop)
2. `dpID 115 = <new current>`
3. `dpID 124 = 0` (Charge)

The car tolerates this pause (equivalent to unplug/replug). dpID 124 does work at any time,
and small changes in **2 A steps** have been reported to work live.

## Firmware

- **Main YAML**: [`feyree-c6.yaml`](feyree-c6.yaml) (XIAO ESP32-C6, recommended).
- **Alternative**: [`feyree-cb3s.yaml`](feyree-cb3s.yaml) (Beken CB3S module on the WBR3 footprint).
- **HA automation example**: [`ha-carga-solar-ejemplo.yaml`](ha-carga-solar-ejemplo.yaml) —
  solar-surplus dynamic charging (stop → set current → start, 2 A steps, hysteresis,
  cooldown). Entity names are examples; yours will carry the MAC suffix.
- `secrets.yaml` contains **only** `wifi_ssid` / `wifi_password` (see
  [`secrets.yaml.example`](secrets.yaml.example)). Trusted home LAN: no web password, no API
  encryption, no OTA password.

Config details:

- `name_add_mac_suffix: true` → unique hostname like `feyree-c6-9f519c.local`.
- `web_server:` enabled (port 80) for control/diagnostics without Home Assistant.
- `captive_portal` + open fallback AP for recovery if WiFi fails.
- Sensors with proper `device_class`/`state_class`. Session energy (112) has **no**
  `state_class` because it resets every session.
- In ESPHome, Tuya selects use **`enum_datapoint`** (not `select_datapoint`).

## Flashing

### Compile

```bash
# WARNING: compile from a path WITHOUT spaces (esphome fails on paths like "Mi Nube")
mkdir -p /tmp/esphome_feyree_c6
cp feyree-c6.yaml secrets.yaml /tmp/esphome_feyree_c6/
esphome compile /tmp/esphome_feyree_c6/feyree-c6.yaml
```

The binary ends up in
`/tmp/esphome_feyree_c6/.esphome/build/feyree-c6/.pioenvs/feyree-c6/firmware.factory.bin`
(plus `firmware.ota.bin` for later OTA updates).

### First flash over USB-C

1. Enter download mode: **hold BOOT while plugging in the USB-C cable**
   (or BOOT + tap RESET). The board appears as `/dev/ttyACM0`
   (`303a:1001 Espressif USB JTAG/serial debug unit`).
2. Flash at offset `0x0`:

```bash
esptool --chip esp32c6 --port /dev/ttyACM0 write-flash 0x0 firmware.factory.bin
```

3. On boot it joins your WiFi and is reachable at `http://feyree-c6-<mac6>.local`.

### OTA updates

```bash
esphome run /tmp/esphome_feyree_c6/feyree-c6.yaml --device feyree-c6-<mac6>.local
```

## Verified status (26-Jul-2026)

- [x] Compiles OK (ESPHome 2026.5.3, esp-idf).
- [x] Flashed over USB-C; WiFi + mDNS + web_server OK.
- [x] TuyaMCU handshake with the GD32 — clean dpID reports, no checksum errors.
- [x] Live monitoring (voltage, current, power, temperature, session energy, state).
- [x] **Current control**: `Setting datapoint 115` confirmed back by the GD32 and **reflected
  on the charger's own display** (tested 16→21→10 A).
- [ ] Full charge start/stop sequence (14/124) with a car plugged in.
- [ ] Solar-surplus dynamic charging in production.

Project currently **paused** (house move); the charger is fully assembled and waiting.

![Final assembly: the XIAO C6 on the lid, wired to the UART header and the 5V footprint](docs/images/final_assembly.jpg)

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Initialization failed at init_state 0` loop | RX/TX swapped, wrong UART, or a dead module still on the footprint | Swap the two wires; use the header, not the RS485 test points; remove the old module |
| Shows up in HA for 1 s then drops | Brownout on WiFi peaks (weak rail / loose contact) | Power from the 5V footprint; check solder joints |
| LED blinking, never solid, offline | Boot loop from unstable power (dupont) | Measure 5V/GND **on the C6's own pads** |
| Ping OK but HTTP 000 | web_server hung after a boot loop | Power-cycle the C6 |
| Sensors arrive but commands don't | Corrupt frames (dupont in our case) | Solder the wires, keep them short |
| Live logs without a PC | — | `curl -sN http://<ip>/events` (web_server SSE log) |

## Lessons learned

- **Dupont jumpers are a lottery. Always solder** permanent wiring. They were the probable
  cause of every control failure we chased for a whole day.
- A rail at **1.2 V boots the chip but can't hold it** (WiFi brownout). Measure the rail
  *before* blaming the firmware.
- "Device not on the network"? Power it over USB with the charger unplugged — that separates
  firmware/WiFi problems from power problems.
- A 230 V slip while probing killed a multimeter and (most likely) the board's 3V3 LDO. The
  charger itself survived. **Mains voltage is unforgiving: if you're not confident probing a
  live board, don't.**
- The GD32 not reporting control dpIDs (115/14/124/18) spontaneously is **normal** at idle;
  but after a valid command it *must* confirm (`Datapoint 115 update`). No confirmation ever
  = corrupt wiring.

## Safety

This device contains **mains voltage (230 V)**. Everything described here is low-voltage
work (3.3/5 V), but the board around it is live when plugged in. Disconnect mains before
soldering, double-check with a multimeter, and never probe live boards if you don't know
exactly what you are doing.

## Stock firmware backup

A full backup of the stock WBR3 firmware was made before desoldering (recommended: do your
own with `ltchiptool flash read ambz2 wbr3_backup.bin -d /dev/ttyUSB0` — note the WBR3 must
be **desoldered** to reach the strap/log pads). It will be published here once located and
verified free of credentials.

## References

- **Feyree teardown (stonacek, Elektroda)** — dpID map, scale factors, charge sequence:
  https://www.elektroda.com/rtvforum/topic4085036.html
- **Official Tuya WBR3 MCU serial schematic** (MCU UART on pads 15/16):
  https://developer.tuya.com/en/docs/iot/wbr3-module-mcu-serial-communication-instructions?id=K9pfgdk62ae6g
- **ESPHome TuyaMCU**: https://esphome.io/components/tuya.html
- **ESPHome Tuya select**: https://esphome.io/components/select/tuya.html
- **Seeed XIAO ESP32-C6 wiki / pinout**: https://wiki.seeedstudio.com/XIAO_ESP32C6_Getting_Started/
- **LibreTiny docs (why the WBR3 is not viable)**: https://docs.libretiny.eu/
- **HA Community thread — Feyree WiFi smart EV charger**:
  https://community.home-assistant.io/t/feyree-wifi-smart-ev-charger/718634
- Community ESPHome Feyree components (not used here, our own YAML):
  https://github.com/vm03/esphome-feyree-ev-charger ·
  https://github.com/m-przybylski/esphome-feyree-ev-charger

## License

AGPL-3.0 — free to use, modify and share under the same terms.
