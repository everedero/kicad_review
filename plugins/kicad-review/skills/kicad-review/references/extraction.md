# Which command answers which question

Every command here was run against real boards. Timings are from a 23 MB, 2000+ footprint
board, so they are an upper bound.

Set up once:

```bash
P=path/to/project
S=$P/<root>.kicad_sch
B=$P/<board>.kicad_pcb
mkdir -p .review-scratch
export PATH="$HOME/.local/bin:$PATH"
```

## Authoritative checks

```bash
# ERC. ~2 s. Writes sheets[].violations[] with type, severity, description, items[].pos
kicad-cli sch erc --format json --severity-all -o .review-scratch/erc.json "$S"

# DRC plus schematic/PCB parity. ~13 s on the 23 MB board.
# Writes violations[], unconnected_items[], schematic_parity[]
kicad-cli pcb drc --format json --severity-all --schematic-parity \
    -o .review-scratch/drc.json "$B"
```

Add `--refill-zones` if zone fills look stale, but never together with `--save-board`: the
design under review must not be modified.

Read the JSON by counting, never by printing:

```bash
python3 -c "
import json; from collections import Counter
d = json.load(open('.review-scratch/drc.json'))
v = d.get('violations', [])
print(Counter((x['type'], x['severity']) for x in v).most_common())
print('parity:', len(d.get('schematic_parity', [])))
"
```

ERC violations are nested one level deeper, under `sheets[].violations[]`.

## Project settings, small enough to read directly

`$P/*.kicad_pro` is JSON. The two fields that matter:

- `net_settings.classes` — the netclass list, with `track_width` and `clearance`
- `erc.rule_severities` — which ERC checks have been downgraded or ignored

A project with only a `Default` netclass fails Eve's netclass policy immediately. So does a
project with one class per net rather than one class per function.

`$P/*.kicad_dru` is the existing custom rule set. Read it: it shows what Eve's team already
decided, and rules should be added to it rather than replacing it.

## Schematic questions

| Question | Command |
|---|---|
| Is the design hierarchical, how deep, how many sheets | `kicad-sch-hierarchy "$S" stats` |
| Sheet tree | `kicad-sch-hierarchy "$S" tree` |
| Are sheet pins and hierarchical labels consistent | `kicad-sch-hierarchy "$S" validate` |
| Which sheets are instantiated more than once | `kicad-sch-hierarchy "$S" list --format json` |
| Every component and net in the whole hierarchy | netlist export, see below |
| Components on one sheet | `kct symbols "$SHEET" --filter "U*"` |
| Nets on one sheet | `kct nets "$SHEET" --net VCC` |
| Unconnected pins on one sheet | `kct sch unconnected "$SHEET"` |
| Wire versus label balance on one sheet | `kct sch wires "$SHEET" --stats` |

`kicad-sch-hierarchy` follows sheet instances from the root sheet, so pass the root
`.kicad_sch`, not a child.

**The `kct sch*` commands do not.** They read exactly the one file given. On a hierarchical
design the root sheet is mostly sheet blocks, so `kct symbols root.kicad_sch` answers
"No symbols found" and `kct nets root.kicad_sch --net GND` answers "not found" — both are
correct answers to the wrong question. Run them per sheet, or use the netlist.

For anything hierarchy-wide, export the netlist. It is flat, authoritative, fast (under a
second even on large designs) and agrees with the board:

```bash
kicad-cli sch export netlist --format kicadsexpr -o .review-scratch/net.net "$S"
```

It contains one `(comp ...)` per component with its sheetpath, value, footprint and
properties, and one `(net ...)` per net listing every pin on it. This is the right source
for "which nets carry power", "is this GPIO named by function", and "do the differential
pair members both exist".

## PCB questions

| Question | Command |
|---|---|
| Size, layer count, footprint/net/trace/via/zone counts | `kicad-pcb-query "$B" summary` |
| Via inventory, for stitching and layer changes | `kicad-pcb-query "$B" vias --format json` |
| Trace inventory, for width changes mid-net | `kicad-pcb-query "$B" traces --format json` |
| Stackup | `kicad-pcb-query "$B" stackup` |
| Pads of one part, for fanout review | `kicad-pcb-query "$B" padmap --ref U1` |
| Net connectivity, per netclass | `kicad-net-status "$B" --by-class --format json` |
| Incomplete nets only | `kicad-net-status "$B" --incomplete --why` |
| Stitching via coverage | `kicad-pcb-stitch "$B" --dry-run --blanket --spacing 4.5` |

`kicad-pcb-stitch` can modify a board. **Always pass `--dry-run`.** Eve's spacing is 4 to
5 mm, so `--spacing 4.5` reports against the middle of her range. `--blanket` is what asks
for grid coverage; without it the command reports nothing to do. The count of vias it
would add is the finding: a board needing 23 more vias is thinly stitched, a board needing
none is already covered.

## What no tool will tell you

These need judgement, from the summary plus targeted queries, and belong in prose:

- whether the guard ring exists and is GND on every layer
- whether a block was worth instantiating
- whether GND faces down and positive supplies face up
- whether a note sits next to the resistor it explains
- whether the silkscreen tells someone how to use the board
- whether the design is finished enough for teardrops to be expected
