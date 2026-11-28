# Backplate models: US and EU, heating and cooling

What differs between the backplate for the US (wires straight to the HVAC equipment) and the one
for the UK/EU (two wires to a Heat Link), and what that means for heating, cooling and for what
happens when the head unit sleeps or dies.

Labels: **observed** = seen in a binary or on a device; **inferred** = concluded from that;
**unknown** = not established. Sources: the three backplate firmware images in the stock file
`/nestlabs/share/bp/data/firmware/nlbpfirmware.plist` (head unit firmware 5.9.x, images of April
2019), static analysis of stock `nlclient`, and two EU units (head unit J49, 2nd generation).
Nothing on this page was checked on a US backplate. Full notes:
[backplate-heatlink-flow.md](https://github.com/wian007/nest-v2-firmware/blob/main/docs/backplate-heatlink-flow.md).

## Three firmware families

Observed in the plist (each entry has a version and the backplate models it is for):

| Name in the file | Version | For backplate models | Size | Region |
|---|---|---|---|---|
| `TFE (BP_D2)` | 4.2.8 | `Backplate-2.0` .. `Backplate-2.20` | 60,464 bytes | US (inferred) |
| `TFE (BP_D2D)` | 4.2.8 | `Backplate-2.20` .. `Backplate-2.30` | 63,296 bytes | US (inferred) |
| `TFE (AMBER_BP)` | 1.0.26 | `Backplate-3.0` .. `Backplate-3.20` | 52,416 bytes | UK/EU, with a Heat Link (observed) |

- `AMBER_BP` is the UK/EU one: both EU units we have report `Backplate-3.2` and run it.
- That `BP_D2` and `BP_D2D` are the US backplates with the spring terminals is inferred: they are
  the other two, and only they hold texts about several circuits ("No suitable circuit",
  "Circuit not enabled", and in `BP_D2D` over-current handling per circuit: "oc_circuits",
  "Circuit %d OFF in pre-charge"). Not checked on hardware.
- All three are for an STM32 with the image at 0x08000000, and each comes with a boot loader
  (`BSL` 3.1).

## One command set

Observed: the three images dispatch the same commands. The table of handled command ids is the
same in `BP_D2` and `AMBER_BP`; `BP_D2D` adds two:

| Image | Handled ids |
|---|---|
| `BP_D2`, `AMBER_BP` | 81 82 83 84 85 86 8f 90 91 92 93 98 99 9b 9c 9d 9e 9f a1 a2 a3 a4 b1 b2 b3 b5 b6 b7 b8 b9 ba c0 c2 c3 |
| `BP_D2D` | the same, plus c4 c5 |

So the frame format, the handshake, quiet mode (`0xa1`, `0xa4`) and the switch command (`0x82`)
are common. What differs is what stands behind the switch command.

## Heating and cooling

The switch command is `0x82 [circuit][state]`; the frame accepts circuits 0..12.

| | UK/EU (`AMBER_BP`, Heat Link) | US (`BP_D2`, `BP_D2D`) |
|---|---|---|
| Circuits that do something | **one**: circuit 0 (observed: the switch function returns at once for every other circuit) | several (inferred from the texts and from `nlclient`) |
| Heating | circuit 0 = call for heat. The backplate signals the Heat Link over T1/T2 and the Heat Link switches the boiler relay | one or more heating stages, each a wire |
| Cooling | **none**. There is no cooling circuit, no fan and no heat pump on this backplate | cooling stages, fan, heat pump reversing valve, auxiliary and emergency heat, humidifier and dehumidifier: `nlclient` keeps a state for each (`Stage1CoolState`, `FanState`, `AuxHeatState`, ...) |
| Hot water | unknown. The Heat Link has a hot-water relay; over T1/T2 we only found the one circuit | not applicable |
| Supply | 12 V DC from the Heat Link over T1/T2. Observed in the state frame `0x000b`: Vin 11.3 V | the HVAC transformer, with a C wire or by power stealing (`0xc0`) |
| Wire detection | the wire state frame is sent and must be acknowledged (`0x8f`), though there are no wires to detect | detects which wires are connected |

**Unknown for the US:** which circuit number is which wire (W1, Y1, G, O/B, ...). `nlclient`
maps HVAC functions to "switch ids", but that table was not read.

## Safety temperature: heat and cool (command 0x89)

The "Safety Temperature" of the Nest menu has a heating side (keep the house above a lower
limit, frost protection) and a cooling side (keep it below an upper limit). Observed in
`nlclient`: `SetSafetyTempSettings` fills 16 bytes and sends them as command `0x89`:

| offset | type | field |
|---|---|---|
| 0 | u16 | enable: bit 0 = heat, bit 1 = cool |
| 2 | s16 | heat: lower safety temperature - 0.39 C, in 0.01 C |
| 4 | s16 | heat: lower safety temperature + 0.39 C, in 0.01 C |
| 6 | u16 | heat circuits, one bit per circuit |
| 8 | s16 | cool: upper safety temperature - 0.56 C, in 0.01 C |
| 10 | s16 | cool: upper safety temperature + 0.56 C, in 0.01 C |
| 12 | u16 | cool circuits, one bit per circuit |
| 14 | u16 | lockout |

The answer `nlclient` expects is `MSG_ST_STATUS` ("status, current, heattime, cooltime, reason,
ocevents, pwrevents"). It sends `0x89` only when two capability flags of the backplate are set.

**Observed: none of the three firmware images handles 0x89.** In all three it goes to the branch
for unknown commands, together with 0x87..0x8e. So with this firmware generation neither the US
nor the EU backplate switches heating or cooling by a temperature of its own; frost protection
is what the head unit does while it runs. Which backplate, if any, does act on `0x89` is
unknown.

## When the head unit is silent: the watchdog (observed for AMBER_BP)

- Every `0xa1` (the quiet message; payload = seconds the head unit wants to sleep) loads a
  countdown with **payload + 1800 seconds**. A payload of 0 counts as 7200.
- A tick once a second counts it down. At zero the backplate **switches the heat off** and then
  **resets the head unit**.
- As long as the head unit has not acknowledged the wire state after its reset, a routine every
  60 seconds switches the heat off under a condition that was not read further.
- Quiet mode itself leaves the switch alone: the heat stays on while the head unit sleeps.

Inferred: a head unit may sleep with the heat on, and a head unit that hangs with the heat on
leaves it on for at most its last quiet time plus 30 minutes. On one EU unit a hung head unit
was restarted by the backplate about every 29 minutes, which fits. Not tried on purpose yet.

Unknown: whether `BP_D2` and `BP_D2D` do the same (the handler of `0xa1` is in their tables, the
code behind it was not read), and what they do with cooling and fan circuits at that moment.

## Voltages (state frame 0x000b)

Observed on an EU unit, with the offsets given on the [Protocol](Protocol.md) page: Vin 11.33 V
(the Heat Link supply), Vop 4.71 V, Vbat 3.65 V; Vbat agreed with the head unit's own ADC to
about 0.03 V. On this supply the battery of the head unit goes down while the head unit is
awake. Inferred: the stock firmware keeps it up by sleeping deeply and often.
