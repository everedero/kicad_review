---
name: kicad-dft-review
description: Use when reviewing a KiCAD design for test, bring-up and debug - testpoints, debug LEDs, reset buttons, debug ports, not-equipped footprints, probe ground clips, connector keying and mounting holes. Triggers on "testpoints", "design for test", "DFT", "bring-up", "debuggability", "can I debug this board".
---

# Design for test, debug and reliability

This is where Eve's rules differ most from every other KiCAD skill. The question is not
"will it manufacture" but **"can this board be brought up and repaired by a human"**.

A board that works perfectly and cannot be probed is a bad board.

```bash
kicad-pcb-query BOARD.kicad_pcb summary
kicad-pcb-query BOARD.kicad_pcb footprints --filter "TP*" --format json
kicad-cli sch export netlist --format kicadsexpr -o .review-scratch/net.net ROOT.kicad_sch
```

Count testpoints against nets. A board with 200 nets and 4 testpoints is a Major finding on
its own, before looking at which nets they are on.

## Testpoints: which signals

Every one of these needs a testpoint. Check the netlist for the net, then the board for a
testpoint footprint on it:

- every **supply voltage**
- **ground**, several, spread across the whole board
- other important **analog voltages**: control lines, DAC outputs
- **all reset lines**
- **fault output pins**
- **JTAG and flashing ports**
- **memory programming ports**: SDIO, QSPI
- **bus signals**: SPI SDIN, SDOUT, CLK, CS, and the equivalent for other buses

Missing reset or fault testpoints are the most expensive omission, because they are what
you need exactly when the board will not start.

## Testpoints: placement

- On the **bottom** layer if only the top is equipped, otherwise the least crowded layer.
- More than **2 mm from tall components**, so a probe cannot hit them.
- More than **2 mm from through-hole pads**, so solder cannot be poured onto them.
- **Do not move testpoints between board revisions.** If a previous revision exists,
  compare: a moved testpoint invalidates a test fixture and a bring-up procedure. This is a
  Major finding and one that no other tool checks.
- A high-speed testpoint needs a **GND testpoint next to it**, or it cannot be measured.

## Stubs and tearing

- **Low-speed signals:** connect the testpoint through a small stub, so tearing the
  testpoint off does not tear up the main track.
- **High-speed signals:** no stubs at all, one straight track. If a testpoint is needed,
  make it small and place it **on the track itself**, accepting the tearing risk. Always put
  a GND testpoint beside it, preferably a small through-hole that fits an oscilloscope
  short ground spring.

These two rules contradict each other by design. Apply the one that matches the signal, and
say which you applied.

## Debug and measurement features

Report each missing item explicitly. They are cheap before fabrication and impossible
after:

- **Power-on LED**, on the input voltage or on the most important rail. Eve typically
  recommends it on the MCU 3V3 line.
- **Debug LED footprint** for blinky, not-equipped 0603, on a GPIO.
- **Reset button** on any MCU with a reset pin, even if not equipped.
- **Debug port**, SWD or JTAG, on any programmable component, **doubled with testpoints**.
  Plus a bootloader path: UART or USB.
- **Through-hole footprints, not equipped, on the main supplies**, so an external supply can
  be soldered on. Including the GND. This is forgotten constantly.
- **Somewhere to clip an oscilloscope ground**: a 2x1 2.54 mm header, a corner of copper, a
  loop testpoint.
- **Thermal sensors**, usually a good idea.
- **Hardware revision resistors**, and testpoints or a debug connector on spare GPIO, if the
  main features leave many GPIO unused.

## Not-equipped footprints are free

Eve's view: NE options cost nothing and buy configurability. Look for the absence of them:

- 0 ohm resistor footprints to shunt a component out
- I2C address configuration resistor options
- alternative supply options
- any set-up resistor or divider that might be re-valued

**Use 0603, not 0402,** for anything whose value could change at bring-up. Soldering a 0402
by hand on a bench at midnight is a punishment. Report 0402 set-up resistors as a Minor
finding.

## Sub-part debugging

If the board is complicated, or has several MCUs or separable sections, there must be a way
to debug each part alone. For example an I2C connection option to drive a peripheral from
another board, or a direct UART connection in case the onboard serial-to-USB chip is the
thing that is broken.

Ask: if sub-part B is dead, can sub-part A still be tested? If the answer is no, it is a
Major finding.

## Faults must go somewhere

Every fault and error output must be connected to something that can read it, preferably an
MCU GPIO. Trace each fault pin in the netlist. A fault pin tied to a pull-up and nothing
else is a fault nobody will ever see.

## Mechanical and connectors

- **No 1-row SMD 2.54 mm connectors.** They are very fragile. This is a Major finding.
- At least **3 mounting holes**, plus **fiducials** and a **layer indicator**.
- For any connector **without keying**:
  - work out what happens if it is plugged in reversed, and say so plainly
  - consider sacrificing a pin to make it keyed
  - make it very clear on the silkscreen what goes where

Reversed-connector analysis is a Blocker when the answer is "the board dies". Do the
analysis rather than noting that keying is absent: whether it matters depends entirely on
what is on the pins.
