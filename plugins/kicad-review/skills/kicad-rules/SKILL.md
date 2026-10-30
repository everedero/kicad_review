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

Eve's classes, by function:

| Class | Applies to | Min track width |
|---|---|---|
| `POWER` | supply nets up to 2 A | 0.25 mm |
| `HIGHPOWER` | supply above 2 A: amplifiers, motors, lighting | 0.4 mm |
| `MIDFREQ` | > 1 MHz square, > 2 MHz sine, or rise time < 400 ns | clearance > 2x width |
| `HIGHFREQ` | rise < 10 ns, > 50 MHz square, > 100 MHz sine | clearance > 3x width, few vias |
| `SENSITIVE` | enables, resets, chip selects, interrupts, faults | kept clear of fast nets |
| `BUS_<INSTANCE>` | the nets of one peripheral instance, see below | exempt from itself |

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

**Buses and sensitive lines.** A bus is the set of nets of *one peripheral instance*:
`SPI0_MOSI` + `SPI0_MISO` + `SPI0_CLK`, all the DDR nets, all the USB nets. Chip selects,
enables and supplies are **not** bus members; they belong in `SENSITIVE`. The distinction
earns its keep in the PCB rules: bus members may run close to each other because they
switch at nearly the same time, while `SENSITIVE` lines must stay away from anything fast.

Assign both with `netclass_patterns` in `.kicad_pro` rather than net by net:

```bash
python3 -c "import json;print(json.load(open('PROJECT.kicad_pro'))['net_settings']['netclass_patterns'])"
```

Two things to know about patterns, both confirmed against KiCAD 10.0.6:

- **They match the full net name including the sheet path.** A pattern of `SPI0_*` matches
  nothing on a hierarchical design, where the net is `/Soc/Peripherals/SPI0_MOSI`. Lead with
  a wildcard or give the full path.
- **Membership is additive.** A net matching two patterns joins both classes, which is what
  lets a net be in `HIGHFREQ` and `BUS_SPI0` at once. Note the consequence: on a multi-class
  net `A.hasExactNetclass('BUS_SPI0')` is **false**, so use `hasNetclass`, not the exact form.

Enumerate bus members one pattern per net instead of globbing `SPI0_*`. KiCAD patterns have
no exclusion syntax, so a glob drags `SPI0_CS` into the bus — exactly the net Eve excludes.

## Step 3: the clearance rules need real numbers

`MIDFREQ` and `HIGHFREQ` want "clearance greater than two or three times the track width".
DRC cannot read the local track width, so the shipped rules hard-code a multiple of an
assumed netclass width: 2x for `MIDFREQ`, 3x for `HIGHFREQ`. **Read the project's actual
netclass width and recompute** before pasting, and say in the report which width was assumed.

**The bus asymmetry.** The 3x figure applies between a high-speed net and everything
*outside* its bus. Inside the bus there is no such requirement. A flat
`A.NetClass == 'HIGHFREQ'` clearance rule therefore reports every intra-bus pair as a
violation, which on a routed bus is a wall of false positives that trains the designer to
ignore DRC.

When the design has buses, swap the flat `HIGHFREQ clearance` rule for the bus-aware one in
`assets/dru/eve-netclasses.kicad_dru`, adding one clause per bus:

```
(rule "HIGHFREQ clearance outside the bus"
  (severity error)
  (condition "A.hasNetclass('HIGHFREQ') && !(A.hasNetclass('BUS_SPI0') && B.hasNetclass('BUS_SPI0'))")
  (constraint clearance (min 0.45mm))
)
```

**Do not keep both rules, and do not try to solve this with rule order.** DRC evaluates a
pair under both role assignments, so a flat rule still matches with A and B swapped.
Last-match-wins ordering gets close but leaks. Measured on a real board carrying 29
intra-bus pairs:

| Approach | Intra-bus violations |
|---|---|
| relaxed intra-bus rule placed first | 31 — no effect |
| relaxed intra-bus rule placed last | 2 — leaks |
| exemption inside the condition | 0 |

All three keep bus-to-outside enforcement at roughly 489, so the exemption costs nothing.

The constant-width rules use `(min X) (max Y)` deliberately: Eve's rule is "do not change
track width mid-course", so a *widened* section is a finding too, not just a necked one.

## Step 4: ERC flags

Place the three flags from `assets/symbols/eve_flags.kicad_sym`. They are **not** power
symbols, and that is deliberate.

**The `(power)` trap.** KiCAD's own `power:PWR_FLAG` carries `(power)` and a pin named `~`,
and attaches to a net without renaming it. That is a special case inside KiCAD, keyed on the
symbol, and copying the idiom does not copy the behaviour. Any *other* symbol carrying
`(power)` acts as a global power label named by its Value field, so every instance of it
shorts together. Verified on asynth2osc against 10.0.6: with `(power)` present, seven
`PWR_IN_FLAG`s merged `VBUS`, `5V_CLEAN`, `IN_+12V`, `VDDA`, `3V3_FILT`, `5V` and `+10V`
into one net, silently — the netlist simply had six fewer nets. Removing `(power)` restored
the net list exactly. Keep these three symbols non-power; the hidden `#FLG` reference and
`in_bom no` / `on_board no` already keep them off the BOM and the board.

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

**What each flag actually raises, measured.** `PWR_IN_FLAG` fires `power_pin_not_driven`
when the rail it sits on has no `power_out` driver reaching it — which is the point, since
drive does not propagate across a passive. `OUTPUT_FLAG` × 2 on one net fires `pin_to_pin`
("Output… Output"). `INPUT_FLAG` alone on a passive-only net raises **nothing** in KiCAD
10.0.6: an `input` pin needs no driver. Confirmed by swapping that pin to `power_in`, which
did fire, so ERC does see the symbol. Treat `INPUT_FLAG` as documentation plus the
counterpart that makes an `OUTPUT_FLAG` conflict meaningful, not as a check in its own right.

Add the library to the project with Preferences > Manage Symbol Libraries, project tab, or
by hand in `sym-lib-table`:

```
(lib (name "eve_flags")(type "KiCad")(uri "${KIPRJMOD}/eve_flags.kicad_sym")(options "")(descr "Eve ERC flags"))
```

**If you ever write flag instances into a `.kicad_sch` by script**, note that the per-sheet
`lib_symbols` cache prefixes only the *top-level* symbol name (`eve_flags:PWR_IN_FLAG`); the
unit sub-symbols stay bare (`PWR_IN_FLAG_0_0`). Prefixing the sub-symbols makes KiCAD load
the whole sheet **empty, with no error and a zero exit code** — the only symptom is a
netlist that lost that sheet's components. Always diff component and net counts against a
pre-edit netlist.

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

**Running DRC is not a validation.** `kicad-cli` accepts a broken `.kicad_dru` in silence.
Verified against 10.0.6: a file with an unbalanced paren, and a file calling a misspelled
`hasNetclassTYPO()`, both reported exactly the same violation count as the correct file,
with no error, no warning and a zero exit code. There is no `rule` or `syntax` violation
type to look for. A rule can also parse perfectly and match nothing, which looks identical.

So prove the rule *fires*, with the absurd-constraint technique:

1. Run DRC with no custom rules to get the **baseline** count.
2. Copy the proposed rule in and widen its constraint until it must be violated —
   `(constraint clearance (min 8mm))`. Re-run.
3. **The count must rise above baseline.** If it does not, the rule is either unparsed or
   matching nothing; both are failures, and the DRC run cannot tell you which.
4. Check the violations name the nets you expect, not just that the number moved. Counts
   saturate — several different conditions all produced exactly 499 clearance violations on
   one test board, so identity is the real evidence.
5. Restore the real constraint value.

A `.kicad_dru` that KiCAD rejects is worse than no rule, because the project looks
protected and is not — and nothing in the tooling will tell you.

Known syntax traps, from Eve's own existing rules:

- `memberOfFootprint('?')` as a wildcard **does not work**. Name footprints in full, or
  match on `A.Reference`.
- `physical_clearance` and `physical_hole_clearance` ignore nets; plain `clearance` does
  not. Pick deliberately.
- The exclusion-then-assertion pair is how to express "there is no X near Y": one rule with
  `(severity ignore)` to absorb the acceptable pairs, a second with
  `(constraint assertion ...)` to report what is left.
- Rules are evaluated last-match-wins per constraint type, so order matters. Do not lean on
  it to carve an exception out of a broader rule: a pair is evaluated under both A/B role
  assignments, so the broad rule matches again with the roles swapped. Put the exception in
  the broad rule's own condition instead.
- `A.hasNetclass('X')` tests membership and is what to use when a net may hold several
  classes. `A.hasExactNetclass('X')` is false as soon as the net joins a second class.
  `A.NetClass == 'X'` still matches a multi-class net in 10.0.6, so the existing rules in
  `eve-netclasses.kicad_dru` are not broken by patterns — but prefer `hasNetclass` for
  anything testing the B side, which `A.NetClass` cannot express.
