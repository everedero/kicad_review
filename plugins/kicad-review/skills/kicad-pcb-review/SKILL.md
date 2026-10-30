---
name: kicad-pcb-review
description: Use when reviewing KiCAD PCB layout - guard ring, stitching vias, isolated copper and PCB antennae, net priorities, crystals, high-speed routing, layer changes, thermal reliefs, length matching, fanout, teardrops, silkscreen and soldermask. Triggers on "review this PCB", "check my layout", "routing review", "stitching vias", "guard ring", "silkscreen".
---

# PCB layout review

Eve's first instruction for this whole area:

> Prefer making custom DRC rules rather than checking PCB manually.
> If DRC rules are not enough, try to suggest a plugin.

So work in this order: switch on the built-in check, then write a custom rule, then suggest
a plugin, and only then review by eye. Hand findings to `kicad-rules` to be turned into
`.kicad_dru` text.

```bash
kicad-cli pcb drc --format json --severity-all --schematic-parity \
    -o .review-scratch/drc.json BOARD.kicad_pcb
kicad-pcb-query BOARD.kicad_pcb summary
kicad-pcb-query BOARD.kicad_pcb stackup
kicad-pcb-stitch BOARD.kicad_pcb --dry-run --blanket --spacing 4.5
```

Count the DRC violations by type and severity. Do not print the report.

## Checks KiCAD already does

Switch these on before writing anything:

| Eve's rule | Built-in violation type |
|---|---|
| PCB antennae | `isolated_copper` |
| Thermal relief not in a real pour | `starved_thermal` |
| Schematic and board disagree | `schematic_parity` |
| Silkscreen over pads, or overlapping | `silk_over_copper`, `silk_overlap` |
| Unroutable or unconnected copper | `unconnected_items` |

An `isolated_copper` count is the PCB antenna count. Report it as such.

## Copper pour and ground

- **Guard ring.** A GND plane or track all around the board, on every layer, respecting the
  edge-to-copper constraint, at least as wide as a via's copper pad. No DRC rule can prove a
  ring exists, because that is a topology claim; check it by eye from a plot, and use the
  edge clearance rule to protect the space it needs.
- **Stitching vias**, roughly on a grid, 4 to 5 mm spacing. `kicad-pcb-stitch --dry-run
  --blanket --spacing 4.5` reports how many vias it would add; that number is the finding.
  Not required where there are already many GND vias or through-hole pads.
  **Avoid stitching over power planes.**
- **PCB antennae** — copper with a via on one side and none on the other. For each
  `isolated_copper` hit, suggest either adding the missing via, or a Rule Area with
  "Keep out zone fills" so the pour never reaches there. Prefer the keep-out when the
  copper serves no purpose.

## Not all nets are equal

Eve's priority order, highest first. Review in this order and spend the effort here:

1. **Crystals and oscillators.** Very close to their chip, **never** change layer, copper
   clearance more than 3x the track width, good GND around, stitching vias. A crystal with a
   via in its loop is a Major finding.
2. **High-speed.** Avoid layer changes. Avoid plane crossings under the track — a split in
   the reference plane under a fast signal breaks the return path. Clearance more than 3x
   the track width, GND all around, stitching vias.
3. **Normal GPIO and low-speed** — enables, faults. These may take indirect paths. Do not
   spend review effort here, and do not report a wandering enable line as a fault.

Supporting queries: `kicad-pcb-query BOARD.kicad_pcb vias --format json` and
`traces --format json`, filtered by net.

## Track geometry

- **No 90 degree angles.** 45 degrees preferred.
- **Do not change track width mid-course** on a signal track, especially mid and high
  speed. Expressible as a netclass rule with both `(min)` and `(max)`.
- **A wide power track needs an equally wide GND return.** If a power net uses a wide track,
  the matching reference GND track must be at least as wide, for the DC return path. Check
  the pair, not just the supply.
- **Fanout on a grid.** For every component over 20 pins, the grid should be twice or four
  times the pin pitch, and tracks should leave the part straight and clean. Surrounding
  passives, or those on the bottom for a BGA, should be regularly spaced with pad centres
  aligned so tracks hit them head on. Check with
  `kicad-pcb-query BOARD.kicad_pcb padmap --ref U1`.

## Placement

Analog, digital and power circuits placed **separately**, not mixed. In particular **no
analog supply routed under a digital chip or a DC-DC converter**. This is a placement
finding, so it is judged from a plot, not from a count.

## Thermal reliefs

Eve's policy is conditional. Do not apply it as a blanket rule:

- **Good** on GND through-hole pads, when the component is soldered by hand.
- **Probably bad** on SMD pads, especially on a design with mid or high speed signals.
- **Useless** if the pad is not in a big copper pour — which is what `starved_thermal`
  detects.

State which case applies before making a recommendation.

## Length matching

- **No wavy length-matching tracks for differential signals** such as USB or CAN. The
  differential routing is match enough, and the wiggles make it **worse, not better**.
  Several other tools recommend the opposite; they are wrong for these buses. Report
  existing serpentines on USB or CAN as a finding.
- **Precise length matching is for DDR and some flash memories only.** Get the tolerance
  from the datasheet. Note that KiCAD does not yet account for internal silicon track
  length, so the numbers in the tool are not the whole budget.

## Finishing touches

Only report these on a design that claims to be final. On an in-progress design, note that
they are pending instead:

- **Teardrops** present. They are one of the last steps.
- Small parts, testpoints and any soldermask opening kept **more than 4 mm from the side
  opposite through-hole connectors**, in case selective wave soldering is used.

## Silkscreen

If the board is assembled by pick-and-place, **do not** put every component identifier on
the silkscreen. It is overcrowded and the machine does not use it. Keep it minimal:

- pin 1 indicators, for anything that is not a basic passive
- **testpoint labels** — `3V3`, `SCL` — which is what makes a testpoint genuinely useful
- connector indications — `Debug USB`, `Main power`
- board name and board version

Size above **0.80 mm height and 0.15 mm stroke**. Manufacturers can go smaller; Eve prefers
silkscreen that is easy to read.

The test: does the silkscreen say what goes where, and give someone a good starting point
for using the board? A board covered in `R47` and `C112` and no labels on its connectors
fails that test while looking thorough.

## Soldermask

**No white or yellow soldermask**, because optical inspection will fail. Check the stackup
and the fabrication notes. This is a Blocker if the board is about to be ordered.
