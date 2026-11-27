# Alternative firmware

Projects that run other software on the 2nd generation thermostat than the stock firmware.

## nest-v2-firmware: a current Linux and a local thermostat (J49)

<https://github.com/wian007/nest-v2-firmware> (MIT; hobby project, no warranty, read its disclaimer)

Not a modification of the stock system but a replacement: its own U-Boot 2024.07, Linux 6.6
with a device tree for the J49 board, a Yocto root file system, and small Rust programs for the
thermostat. No cloud; it is reached over the home network.

| Part | What it does |
|---|---|
| `bpd` | talks to the backplate (the protocol on these wiki pages) and is the control loop: hysteresis, minimum on/off times, frost protection, week schedule, away |
| `j49-screen` | the round panel, the ring, the piezo: a dial like the thermostat card of Home Assistant |
| `j49-web` | a web page on the device; login by confirming a code on the thermostat with the ring |
| `j49-mqtt` | MQTT with Home Assistant discovery |
| `j49-sleep` | suspends when idle; the backplate and the WiFi chip (wake-on-WLAN) wake it |
| installer | a page for Chrome/Edge that does the install over USB (WebUSB): full NAND backup, boot loader, system, and back to stock |

State at the time of writing (November 2026): it controls a Heat Link on a test backplate and
did so on a real one for a night; the battery did not hold on the real backplate with the head
unit awake, so sleeping has to be proven before it can stay on a wall. The installer page is
new and not yet tried on a thermostat. The repository says per document what was checked.

Things from that work that may be useful for other projects, written up on this wiki:

- [Heating control](../backplate/Heating%20control.md): the switch command `0x82`, its answer
  `0x0006`, and why `0x8f` must echo the wire state.
- [Sleep and wake-up](../backplate/Sleep%20and%20wake.md): quiet mode, the wake-up mask, the
  wake-up signal.

And in the repository itself:

- [Flash layout and ECC formats](https://github.com/wian007/nest-v2-firmware/blob/main/docs/flash-layout.md):
  which partition is read by which loader with which ECC.
- [The 802.15.4 radio (EM357)](https://github.com/wian007/nest-v2-firmware/blob/main/docs/em357-radio.md):
  wiring, Nest's SPI bootloader protocol, what runs on the chip.
- [Hardware notes](https://github.com/wian007/nest-v2-firmware/tree/main/docs/hw): clocks, GPIO, pin mux
  as the stock system sets them.
- [j49-emulator](https://github.com/wian007/j49-emulator): the screen program on a PC.
