---
layout: default
title: Documentation for Revision A of the Reader board
---

# Revision A of the Reader board

## Electrical characteristics

(To be determined)

## Soldering guide

This is a recommended soldering procedure. It is designed to help find problems with the board as soon as possible in as local area as possible without wasting components.

Standing recommendation is to check all connections for unintended short-circuits or open-circuits and fix them as soon as possible. Also some steps have some recommended checks to perform. If the board fails those tests the problem must be debugged, and debugging cannot be sufficiently proceduralized to form a manual, therefore it's up to the creativity of the one soldering the board.

### Steps

#### Solder the Power Supply unit

Solder components `IC3`, `C6`, `D2`, `C7`, `D3`, `D4`, `L1`, `R11`, `R12`, `C8` and `C9`. Check for shorts on the power supply lines. Then connect +5V power source to the PSU and verify that

  - the output voltage is `3.3V +- 0.2V`
  - idle current through the PSU is less than `5mA`
  - output voltage is stable (with margin of `+- 0.3V` even if the PSU is loaded with a `330 Ohm` resistor
  - output voltage is stable through the entire allowable range of input voltages

#### Solder the MCU

Solder components `IC2`, `C1`, `C2`, `C3`, `R3`, `D1`, `R4`, `R6`, `JP1`, `JP2` and `JP3`. Use STM32F052 for Reader revA and STM32F072 for Reader+ revA. Connect +5V power source to the PSU and SWD debugger and verify that

  - current through the board is less than `30mA`
  - the MCU responds to the debugger

#### Solder the rest of the board

(Note: If soldering Reader revA (not Reader**+** revA) leave out components `D7`, `R13`, `R14`, `R15` and `X1`)

Connect +5V power source to the PSU, SWD debugger and communication interface and verify that

  - current through the board is less than `100mA`
  - all tests form `hw-testing` repository file `tests/test-reader-revA.py` pass
