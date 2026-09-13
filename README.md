# 24-Hour Alarm Clock on an FPGA

Digital timekeeping, alarm set/trigger/clear, and a demo mode, in SystemVerilog on a Terasic DE0-CV.

**Stack:** SystemVerilog · Cyclone V (DE0-CV) · Quartus Prime 22.1

## Highlights

- **Whole design is one counter module** instantiated at six different moduli using a 50 MHz oscillator.
- Counter takes a *modulus* and derives its own width: `#(parameter int m = 13, parameter int b = $clog2(m))`.
- Alarm is a second, parallel counter chain that will only advance while being set. A 4-digit comparator drives the alarm latch.
- Hours stored as a flat 0–23 count, split into digits at the last moment with two comparators, at a cheaper cost than carrying BCD.

## Counter chain

| Counter | Modulus | Rolls into |
|---|---|---|
| Minute tick | 3,000,000,000 (60 s @ 50 MHz) | Minutes-ones |
| Minutes ones | 10 | Minutes tens |
| Minutes tens | 6 | Hours |
| Hours | 24 | — |
| Half-second | 25,000,000 | Button repeat rate, demo mode |
| Second | 50,000,000 | LED9 heartbeat |

## Controls

| Control | Function |
|---|---|
| `SW0` | Reset. Current time and alarm → 00:00. |
| `SW1` | Set current time. Clock freezes. |
| `SW2` | Set alarm. Clock keeps running; display shows alarm time. |
| `SW4` | Demo mode. Time runs 120× faster. |
| `KEY0` | Silence alarm, reset alarm to 00:00. |
| `KEY2` | Increment minutes |
| `KEY3` | Increment hours |

`HEX3:HEX2` hours, `HEX1:HEX0` minutes. `LED9` blinks 1 Hz. `LED7` blinks on alarm.

`SW1`/`SW2` and `KEY2`/`KEY3` are each mutually exclusive — activating both does nothing.

## Files

| File | Purpose |
|---|---|
| `alarm_clock.sv` | Counter chains, set logic, digit split, alarm comparator, display mux. |
| `counter.sv` | Parameterized modulo counter, `$clog2` width. |
| `ourDff.sv` / `ourHex.sv` | D flip-flop with sync reset + enable; BCD → 7-segment. |

## Build

1. Open `alarm_clock.qpf` in Quartus Prime 22.1
2. Compile, program `.sof` to a DE0-CV
3. For testing, turn on `SW4` first to run the alarm clock 120x faster

## Limitations

- **Alarm can re-trigger at midnight.** Clearing resets alarm time to 00:00, so the comparator matches again when the clock rolls over. A separate "armed" flip-flop would fix it.
- No seconds display, and no way to set seconds.
