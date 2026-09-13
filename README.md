# 24-Hour Alarm Clock on an FPGA

A working alarm clock in SystemVerilog, running on a Terasic DE0-CV. It keeps time, lets you set both the current time and an alarm, and blinks an LED when the two match.

## How it keeps time

The board has a 50 MHz oscillator and nothing else — no RTC, no crystal divider, no help. Everything is built out of a single parameterized counter module that takes a modulus and derives its own width:

```systemverilog
module counter #(parameter int m = 13, parameter int b = $clog2(m)) ...
```

That one module, instantiated with the right modulus, is the whole design. A mod-3,000,000,000 counter rolls over once a minute at 50 MHz and drives a mod-10 minutes-ones counter. That rolls into a mod-6 minutes-tens counter, which rolls into a mod-24 hours counter. Seconds never exist as a value anywhere — the design just counts clock cycles until a minute has gone by.

Hours are stored as a plain 0–23 count and split into digits at the last moment, by comparing against 19 and 9 and subtracting 20 or 10. That's cheaper than carrying BCD around and it's only two comparators.

The alarm is a second, completely parallel copy of the same counter chain that doesn't advance on its own — only when you're setting it. A comparator watches both chains, and when all four digits agree it sets a latch that drives `LED7`. `KEY0` clears the latch and resets the alarm back to 00:00.

`SW4` is a demo mode that swaps the minute tick from the 1-minute counter to the half-second counter, running the clock 120× faster so you don't have to wait an hour to test the alarm.

Button presses are gated on the half-second counter rolling over, which means a held key increments twice a second and a mechanical bounce can't register twice. It's a crude debounce, but for a clock it's the right one — you want a slow, deliberate repeat rate anyway.

## Controls

| Control | Function |
|---|---|
| `SW0` | Reset. Clears current time and alarm to 00:00. |
| `SW1` | Set current time. The clock freezes while this is on. |
| `SW2` | Set alarm. The clock keeps running underneath, and the display shows the alarm time. |
| `SW4` | Demo mode — time runs 120× faster. |
| `KEY0` | Silence the alarm and reset the alarm time to 00:00. |
| `KEY2` / `KEY3` | Increment, in whichever mode is active. **See the note below — verify which is which on your board.** |

`HEX3:HEX2` shows hours, `HEX1:HEX0` shows minutes. `LED9` blinks once a second so you can tell the clock is alive. `LED7` blinks when the alarm is going off.

Only turn on one of `SW1` / `SW2` at a time, and only press one of `KEY2` / `KEY3` at a time. The logic explicitly requires the other to be inactive, so pressing both does nothing.

## What's in here

| File | What it does |
|---|---|
| `alarm_clock.sv` | Everything — counter chains, set logic, digit splitting, alarm comparator, display muxing. |
| `counter.sv` | Parameterized modulo counter. Takes the modulus, computes its own width. |
| `ourDff.sv` | D flip-flop with synchronous reset and enable. |
| `ourHex.sv` | BCD to seven-segment decoder. |

## Running it

1. Open `alarm_clock.qpf` in Quartus Prime (built with 22.1).
2. Compile and program the `.sof` onto a DE0-CV.
3. Turn on `SW4` first — otherwise testing the alarm takes real time.

## Known rough edges

- **The `KEY2` / `KEY3` labels above may be backwards.** The DE0-CV's keys are active-low, but the set logic treats them as active-high (`KEY2 && !KEY3`), while `KEY0` is correctly handled as active-low. So the two halves of the design disagree about button polarity. The hardware behaves consistently either way — but confirm on the board which key actually does hours and which does minutes, and fix either the labels or the logic.
- **The alarm can re-trigger at midnight.** Clearing it resets the alarm time to 00:00, so if the clock later reaches 00:00 the comparator matches again.
- There's no seconds display and no way to set seconds — the minute counter starts wherever reset left it.
