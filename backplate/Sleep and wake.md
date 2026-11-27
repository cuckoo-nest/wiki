# Sleep and wake-up (quiet mode)

How a sleeping head unit and the backplate work together. From the backplate firmware
`AMBER_BP 1.0.26` (disassembly), stock `nlclient` / `nlsleepd` (5.9.x), and tests on a J49 with
our own software. **Observed** / **inferred** / **unknown** as on the other pages. Full notes:
[sleep-j49.md](https://github.com/wian007/nest-v2-firmware/blob/main/docs/sleep-j49.md),
[sleep-stock.md](https://github.com/wian007/nest-v2-firmware/blob/main/docs/sleep-stock.md).

## The two commands

| id | payload | what it does |
|---|---|---|
| `0x00a4` | u16 mask | "set wakeup mask": the events on which the backplate may wake the head unit. It only stores the mask. |
| `0x00a1` | u16 seconds | "quiet message (max_sleep_seconds)": the backplate goes quiet: no stream, it buffers. No payload or 0 means 7200 s. It also reloads the watchdog the backplate keeps on the head unit (seconds + 1800). |

**Every valid frame from the head unit ends the quiet mode** (observed in the receive path of
the firmware and on the device). So `0xa1` has to be the last frame before the head unit
sleeps. A client that sends `0xa1` together with its `0x83` keep-alive never gets a quiet
backplate.

Mask bits, from the names `nlclient` logs for them:

| bit | name | | bit | name |
|---|---|---|---|---|
| `0x0001` | TIMER | | `0x0040` | HUMIDITY |
| `0x0002` | BUFFERS_FULL | | `0x0080` | NEAR_PIR (or TP, by model) |
| `0x0004` | TEMPERATURE | | `0x0100` | VERGENCE |
| `0x0008` | PIR | | `0x0200` | VBAT |
| `0x0010` | PROX | | `0x0400` | ERRORS |
| `0x0020` | ALS | | | |

Stock starts from `0x0005` (TIMER | TEMPERATURE) and adds PIR, VBAT, NEAR_PIR and PROX by
configuration.

## The wake-up signal

The backplate does not wake the head unit with a frame. When the time is over, or an event in
the mask happens, it sends **single `0x00` bytes, about 60 per second**, until the head unit
speaks (observed: 1227 of them in 20 s). On the J49 the serial line is a wake-up source of the
processor, so the first byte ends the suspend.

After waking, `0xa2` gets the buffered sensor data at once (`0x0002`, `0x0022`, `0x0023`,
`0x000c`, `0x0038`, `0x002f`) and the stream starts again; no new handshake is needed.

Tests on the device (our own daemon sends `0xa4`, then `0xa1`, then nothing):

| test | result |
|---|---|
| mask `0x0001`, 20 s, head unit awake | no byte for 19.5 s, then the run of `0x00` bytes |
| the same, then `0xa2` | buffers delivered at once, stream runs again |
| mask `0x0001`, 40 s, then suspend to RAM | asleep 38 s, woken by the serial wake-up interrupt, everything ran on |

## The stock head unit, in short

- `nlsleepd` is a reference-counted suspend manager: programs hold a reference while they need
  the head unit awake; at zero it suspends to RAM.
- WiFi stays associated. The WiFi chip filters packets itself and wakes the processor for three
  patterns only, and it sends TCP keep-alives on the cloud connection by itself (interval
  120 s). That is how a change in the app reaches a sleeping thermostat.
- The backplate wakes it for the timer, temperature, motion and the rest of the mask.

## Unknown

- Whether the other mask bits (PIR, temperature, ...) wake as the timer does: only the timer was tested on its own.
- How long the buffers of the backplate hold.
- The bit of the hall-effect event (`HE` is logged without a visible bit test).
- `0x8b` ("MCU mode" in `nlclient`) is not handled by this backplate firmware at all.
- What the head unit must do so that a real backplate can keep its battery charged: with our
  own software awake all the time, a real EU backplate could not keep up (the battery went from
  3.5 V to under 3.0 V in one night). Stock sleeps almost always.
