---
name: kicad-rules
description: Use when reviewing or setting up KiCAD netclasses, custom DRC rules (.kicad_dru), ERC severities or ERC flag symbols. Covers Eve's POWER/HIGHPOWER/MIDFREQ/HIGHFREQ policy and the PWR_IN_FLAG, INPUT and OUTPUT flags. Triggers on "netclass", "DRC rule", "ERC rule", "kicad_dru", "track width rule", "power netclass".
---

# Netclasses, DRC rules and ERC flags

This is the core skill. Eve's position is explicit:

> Setting up an ERC rule and flags to avoid an issue is preferred over creating a skill.
> Prefer making custom DRC rules rather than checking PCB manually.

So findings from the other skills should land here, converted into rules.

Assets to copy from, in this plugin:

- `assets/dru/eve-netclasses.kicad_dru` — netclass width, clearance and via rules
- `assets/dru/eve-geometry.kicad_dru` — edge clearance, vias, assembly, testpoints
- `assets/symbols/eve_flags.kicad_sym` — `PWR_IN_FLAG`, `INPUT_FLAG`, `OUTPUT_FLAG`

All three are checked against `kicad-cli` and parse cleanly. KiCAD has no include
directive for `.kicad_dru`, so the rules must be pasted into the project's own file,
appended to whatever is already there.

## Step 1: read what exists

```bash
python3 -c "
import json; d = json.load(open('PROJECT.kicad_pro'))
for c in d['net_settings']['classes']:
    print(c['name'], c.get('track_width'), c.get('clearance'))
print('ERC overrides:', d.get('erc', {}).get('rule_severities', {}))
"
cat PROJECT.kicad_dru 2>/dev/null
```

Never delete existing rules. Eve's team writes good ones, and a commented-out rule is
usually a decision, not an oversight.

## Step 2: judge the netclasses

Eve's four classes, by function:

| Class | Applies to | Min track width |
|---|---|---|
| `POWER` | supply nets up to 2 A | 0.25 mm |
| `HIGHPOWER` | supply above 2 A: amplifiers, motors, lighting | 0.4 mm |
| `MIDFREQ` | > 1 MHz square, > 2 MHz sine, or rise time < 400 ns | clearance > 3x width |
| `HIGHFREQ` | rise < 10 ns, > 50 MHz square, > 100 MHz sine | clearance > 3x width, few vias |

Two failure modes, both seen on real boards:

**Only `Default`.** Every net treated alike. This is the common case and a Major finding.
Get the supply nets from the netlist and propose the assignment.

**One class per net.** Classes named `P3V3_OUT_A`, `M12V_VOX` and so on, each with the same
width. The widths may be right but the policy is unmaintainable: a change of mind means
editing twenty classes, and nothing records *why* a net is wide. Propose collapsing them
into `POWER` and `HIGHPOWER` by current, keeping per-net classes only where a net genuinely
needs a different width from its peers.

To find the supply nets, read the netlist rather than guessing from names:

```bash
kicad-cli sch export netlist --format kicadsexpr -o .review-scratch/net.net ROOT.kicad_sch
```

A net is a supply net if power symbols or regulator output pins sit on it. Judge current
from the regulator and the load, and say which number you assumed — the 2 A boundary
between `POWER` and `HIGHPOWER` is a decision Eve must confirm, not something to guess
silently.

Check assignment as well as existence:

```bash
kicad-net-status BOARD.kicad_pcb --by-class --format json
```

A `POWER` class that exists but has no nets in it is worse than no class, because it looks
done.

## Step 3: the clearance rules need real numbers

`MIDFREQ` and `HIGHFREQ` want "clearance greater than three times the track width". DRC
cannot read the local track width, so the shipped rules hard-code 3x an assumed netclass
width. **Read the project's actual netclass width and recompute** before pasting, and say
in the report which width was assumed.

The constant-width rules use `(min X) (max Y)` deliberately: Eve's rule is "do not change
track width mid-course", so a *widened* section is a finding too, not just a necked one.

## Step 4: ERC flags

Place the three flags from `assets/symbols/eve_flags.kicad_sym`. They are power symbols
whose pin name is `~`, which, exactly like KiCAD's own `PWR_FLAG`, means they attach to a
net without renaming it.

**`PWR_FLAG` placement.** Eve's rule is that it belongs close to where power actually comes
from: a connector pin, or a passive sitting in series in the power path such as a coil,
shunt, fuse or ferrite. A `PWR_FLAG` parked on a random corner of a rail is a smell: it
silences ERC instead of documenting the source.

**`PWR_IN_FLAG`.** Extends the power-path check in the other direction. `PWR_FLAG` tells
ERC a net is driven; `PWR_IN_FLAG` tells ERC a net is consumed. Place it downstream of a
series passive so that a rail which is generated but feeds nothing gets reported. Without
it, a fuse with nothing after it passes ERC silently.

**`INPUT_FLAG` and `OUTPUT_FLAG`.** Direction at a boundary. Two `OUTPUT_FLAG`s on one net
is an output driving an output, which is how an uncrossed TX/TX pair is caught before
someone reworks a board. Place them on UART, and on any point-to-point signal crossing a
connector.

Do **not** place them on bidirectional nets. SDA and SCL are shared and must not be
crossed; flagging them as input or output produces false errors. This asymmetry — TX/RX
crossed, SCL/SDA not — is the whole reason the flags exist.

Add the library to the project with Preferences > Manage Symbol Libraries, project tab.

## Step 5: ERC severities

```bash
python3 -c "
import json; print(json.load(open('PROJECT.kicad_pro'))['erc']['rule_severities'])
"
```

Report any check downgraded to `ignore`, and ask why. Thirty-odd overrides is normal on a
mature project; what matters is whether `pin_to_pin`, `power_pin_not_driven` or
`no_connect_connected` are among them, because those are the ones that catch wiring faults.

An ERC warning that everyone has learned to scroll past is the same as no check at all. If
a check is noisy for a good reason, suppress it precisely with a rule rather than globally
with a severity.

## Checking your own rules

Any rule proposed must parse. Verify before putting it in the report:

```bash
mkdir -p .review-scratch/dru-test
cp BOARD.kicad_pcb BOARD.kicad_pro .review-scratch/dru-test/
cp proposed.kicad_dru .review-scratch/dru-test/BOARD.kicad_dru
kicad-cli pcb drc --format json -o /tmp/t.json .review-scratch/dru-test/BOARD.kicad_pcb
```

Then confirm no violation has a type mentioning rule or syntax. A `.kicad_dru` that KiCAD
rejects is worse than no rule, because the project looks protected and is not.

Known syntax traps, from Eve's own existing rules:

- `memberOfFootprint('?')` as a wildcard **does not work**. Name footprints in full, or
  match on `A.Reference`.
- `physical_clearance` and `physical_hole_clearance` ignore nets; plain `clearance` does
  not. Pick deliberately.
- The exclusion-then-assertion pair is how to express "there is no X near Y": one rule with
  `(severity ignore)` to absorb the acceptable pairs, a second with
  `(constraint assertion ...)` to report what is left.
- Rules are evaluated last-match-wins per constraint type, so order matters.
