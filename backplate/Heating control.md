# Heating control (switch command, Heat Link)

How the head unit switches heat through the backplate. Worked out on an EU unit: backplate
"Backplate-3.2" with firmware `AMBER_BP 1.0.26`, connected to a Heat Link over the two wires
T1/T2, head unit J49 (2nd generation). Labels used below: **observed** = seen in a binary or on
the serial line; **inferred** = concluded from that; **unknown** = not established.

Sources: static analysis of stock `nlclient` (5.9.x) and of the backplate firmware image
(`nlbpfirmware.plist`), a serial capture of `nlclient` with ttyproxy, and tests with our own
daemon on the device. The full notes are in the nest-v2-firmware repository:
[backplate-heatlink-flow.md](https://github.com/wian007/nest-v2-firmware/blob/main/docs/backplate-heatlink-flow.md)
and [backplate-protocol.md](https://github.com/wian007/nest-v2-firmware/blob/main/docs/backplate-protocol.md).

## The command

| direction | id | payload | meaning |
|---|---|---|---|
| head unit → backplate | `0x0082` | 2 bytes: `[circuit][state]` | "circuit change request": switch a circuit on (state ≠ 0) or off (0) |
| backplate → head unit | `0x0006` | 2 bytes: `[circuit][state now]` | `MSG_ACK_SWITCH`: the backplate took the command |

For the Heat Link backplate: **`0x82 00 01` = heat on, `0x82 00 00` = heat off** (observed: sent
by our own daemon, acknowledged with `0x0006 00 01` / `0x0006 00 00`, and the relay of the Heat
Link clicks).

What the firmware does with it (observed in the disassembly):

- The handler accepts `0x82` only with exactly 2 payload bytes and only **after the wire ACK**
  (`0x8f`, see below): a flag is set in the `0x8f` handler and cleared on reset. A switch command
  before that is ignored without an answer.
- On this backplate only **circuit 0** does anything; the switch function returns at once for
  every other circuit number (0..12 are accepted).
- Asking for the state it already has changes nothing.
- Switching reprograms a timer output and a DAC: a continuous pulse train. Defaults set at
  start-up are logged by the firmware as `period 60000 ms, pulse 10 ms, guard 50 ms` and
  `100 mA / 8 mA`; the period used for "on" is computed from 1000 ms.

Inferred from that: over T1/T2 the backplate signals the Heat Link with **current pulses on the
12 V supply**, one 10 ms pulse per second while heat is called for and one per 60 s otherwise.
That would make it fail-safe (no pulses, no heat). None of this was measured on the wires.

Observed on the device: the relay click comes **later** than the `0x0006` answer. The answer
only says that the backplate accepted the command, not that the Heat Link switched. No message
that reports the state of the Heat Link itself is known.

## Two names in the reset burst, corrected

In `nlclient` the 13-byte message `0x0004` of the reset burst is the **wire state**
(`MSG_WIRES`, one byte per wire/circuit) and `0x008f` is the **wire ACK**; they are not about
FETs. Two things follow that matter in practice:

- **`0x8f` must carry the 13 bytes of `0x0004`, echoed back.** With an empty `0x8f` the
  backplate answers the identity queries but never starts its sensors, and it ignores `0x82`.
  This was the reason an otherwise correct handshake got no periodic data. (Stock also sends
  `0x85` and a `0xc0` before the reset; which of the two differences is the decisive one for the
  sensors was not isolated. For `0x82` the firmware shows it is the `0x8f` handler.)
- Byte 0 of the wire state was 1 on the backplate with a Heat Link and 0 on a bare backplate on
  the bench (observed, one unit each).

The sequence stock sends, from the capture:

```
HU> c0 00000000          BP> 0014 0000, 000a
HU> 85                   BP> 0017 0000
HU> 83
HU> ff                   BP> 0001 "1.0.26 ...", 0004 <13 bytes>, 0009, 0001 "*sense ...", 0001 "BRK"
HU> 8f <the 13 bytes of 0004, echoed back>
HU> 83
HU> 90                   BP> 0010
HU> 98 99 9d 9b 9c 9f 9e BP> 0018 0019 001d 001b 001c 001f 001e   (one at a time)
HU> c0 00000000          BP> 000a, 0007, 0002 (first temperature/humidity frame)
HU> a2                   BP> 0022 0023 000c 002f
HU> a3
```

## What stock does around it

Observed log strings in `nlclient`:

- A periodic tick; the backplate is reset when it stops answering.
- "HU and BP power state mismatch ... resetting the BP": the head unit compares what it asked
  for with what the backplate reports.
- "Clearing all switches" on a reset or disconnect.
- Safety temperature settings are sent to the backplate ("SetSafetyTempSettings Enable ... Heat
  ... Cool ... lockout"). Inferred: the backplate can switch heating by itself when the head
  unit is asleep or dead. Not looked into further.

`nlclient` also has a separate "heatlink controller" that never calls the backplate send
function and logs OpenTherm registers. Inferred: that is the wireless path (802.15.4) to a Heat
Link, including everything OpenTherm; with only T1/T2 connected, heat is a plain backplate
switch command.

## Unknown

- The delay between the command and the relay, and whether "off" is as fast as "on".
- The meaning of the two current levels and of the 50 ms "guard".
- What `0x0005`, `0x0014` and `0x0017` carry.
