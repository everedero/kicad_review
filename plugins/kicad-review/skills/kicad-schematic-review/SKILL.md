---
name: kicad-schematic-review
description: Use when reviewing KiCAD schematics for hierarchy, sheet layout, drawing conventions, net labels, signal naming, differential pairs, component values and note placement. Triggers on "review this schematic", "check my schematic", "hierarchical sheets", "schematic conventions".
---

# Schematic review

Eve's schematic rules, in review order. Every item here maps to the Schematics section of
`EVE.md`.

Get the structure first, and never read the sheets wholesale:

```bash
kicad-sch-hierarchy ROOT.kicad_sch stats
kicad-sch-hierarchy ROOT.kicad_sch tree
kicad-sch-hierarchy ROOT.kicad_sch validate
kicad-cli sch export netlist --format kicadsexpr -o .review-scratch/net.net ROOT.kicad_sch
```

Remember that `kct symbols` and `kct nets` read one file only. For anything spanning the
design, use the netlist.

## Hierarchy

- **Hierarchical sheets are required.** A flat design of any size is a Major finding.
  `kicad-sch-hierarchy stats` gives sheet count and maximum depth; depth 1 with many
  components means flat in all but name.
- **Repeated blocks must be the same sheet, instantiated.** Two sheets with near-identical
  content are a copy-paste, and they will drift. `kicad-sch-hierarchy list` shows which
  sheet files are used more than once. Near-identical sheet sizes with different filenames
  are the signature to look for.
- **An instantiated block should be worth it**: at least about 10 components or component
  units. Instantiating a block of three resistors adds navigation cost and saves nothing.
- **Repeated functions should be multi-unit components.** Resistor arrays, diode arrays, a
  chip with four identical channels: use units, and put the shared pins (supplies, setup
  pins) on the parent sheet. This matters most in instantiated designs, where a per-channel
  symbol otherwise multiplies.
- Sheet size A4, or A3 when the main component justifies it.
- **No architecture synoptic.** Do not ask for one, and do not report its absence. A block
  diagram belongs in the documentation, not the schematic.

## Drawing conventions

These cannot be extracted by any tool; they need the schematic rendered. Export to PDF or
SVG per sheet and look, rather than parsing coordinates:

```bash
kicad-cli sch export pdf -o .review-scratch/sch.pdf ROOT.kicad_sch
```

- GND and negative supplies face **down**; positive supplies face **up**.
- Pull-ups and pull-downs drawn **vertically**.
- Inputs on the **left**, outputs on the **right**.
- Not too many net labels where a wire would do. A sheet that is all labels and no wires
  hides its own topology. `kct sch wires SHEET --stats` against the label count gives a
  rough ratio; a sheet with many labels and almost no wires deserves a look.

## Information in the right place

- Decoupling capacitors drawn next to the component they decouple, not collected in a
  corner.
- Notes about how a resistor value was chosen sit next to that resistor: divider bridges,
  configuration resistors, current-set resistors.
- Notes must state the applicable **voltage and current ranges** for each power net. A
  power net with no stated range is a Major finding: it is the number everyone needs at
  bring-up and nobody can recover later.

## Naming

Check these against the netlist, where every net and its pins are listed flat.

- **Differential pairs share a prefix and end in `_P` and `_N`**, consistently, through the
  whole hierarchy. A pair that is `_P`/`_N` on one sheet and `+`/`-` on the next will not
  be recognised by KiCAD's differential routing at all. Search the netlist for `_P` and
  confirm each has a matching `_N`, and vice versa.
- **GPIO named by function, not by pin.** `I2C0_SCL`, not `PA10`. Where no function is
  assigned, use the logical pin name for an MCU, and the pin or ball name for an FPGA.
  A netlist full of port names is a Minor finding that becomes Major at bring-up, because
  nothing in the schematic says what the signal is for.

## Component values

- **Standard passive values only.** A netlist with E96 values on an ordinary divider means
  someone computed instead of choosing, and the part may not be stockable.
- **Resistor tolerance other than 1%** must be deliberate. Report any 5% or 0.1% part and
  ask whether it is intended.
- **Capacitor voltage ratings** must be stated and adequate. A capacitor with no voltage
  rating in its value or properties is a finding; derating matters, and a 6.3 V part on a
  5 V rail is a failure waiting for a transient.

Pull values from the netlist `(comp ...)` entries, which carry value and properties per
component with its sheetpath.

## What to hand to kicad-rules

Anything above that can become a check:

- differential pair naming → an ERC-visible naming convention, or a script in CI
- TX/RX crossing → `INPUT_FLAG` and `OUTPUT_FLAG`
- power path completeness → `PWR_FLAG` placement and `PWR_IN_FLAG`
- unconnected pins and pin conflicts → ERC severities, not prose
