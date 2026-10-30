---
name: kicad-review
description: Use when reviewing a KiCAD project - schematics, PCB layout, netclasses, ERC/DRC rules, design for test. Entry point that checks scope, sets up tooling, runs the extraction commands and routes to the domain review skills. Triggers on "review this board", "review my schematic", "check this KiCAD project", "PCB review", "relecture de carte".
---

# KiCAD design review

Review a KiCAD project against Eve's rules. This skill reviews. It never generates
schematics, never autoroutes, and never edits the design under review.

## The one rule that shapes every other

Eve prefers a rule over a comment:

> Setting up an ERC rule and flags to avoid an issue is preferred over creating a skill.
> Prefer making custom DRC rules rather than checking PCB manually.

So for every finding, ask: can this be a `.kicad_dru` rule, an ERC severity change, or an
ERC flag? If yes, the finding is the rule. Give the text to paste, not a lecture. Only
findings that genuinely cannot be mechanised belong in prose.

Many of Eve's rules are already built into KiCAD and only need switching on:

| Eve's rule | Built-in check |
|---|---|
| PCB antennae (copper with a via on one side only) | `isolated_copper` |
| Thermal reliefs that are not in a real pour | `starved_thermal` |
| Schematic and PCB disagree | `kicad-cli pcb drc --schematic-parity` |
| Silkscreen unreadable over pads | `silk_over_copper`, `silk_overlap` |

Check these before writing anything custom.

## Step 1: scope guard

Read `README.md` in the review_skills repo if present. This skill covers interconnection
boards and digital boards (MCU or digital ICs): schematics, design justification, ERC and
DRC rules, netclasses, basic routing sanity, design for test, and PCB layout.

Stop and tell Eve before going further if the board is mainly:

- analog or mixed-signal where the analog part is the point
- RF, or any signal above 50 MHz
- high voltage, above 35 V DC or AC
- battery powered, or containing a charger

Do not silently review these. Say which parts are out of scope, then review the parts that
are in scope. A board with some analog on it is fine; an analog board is not.

Out of scope is not the same as unhelpful. When the analog or mixed-signal part is the
reason the guard fired, say so, review the digital and interconnection parts in full, and
point at the companion in Step 5b for the analog part — it has op-amp, filter and crystal
detectors that this plugin deliberately does not. That is a pointer, not a review: the
analog verdict stays Eve's.

## Step 2: tooling check

Run the doctor before extracting anything, so a missing tool is a one line fix rather than
a failure halfway through:

```bash
kicad-cli --version          # required, 9.0 or later
kct --version                # kicad-tools, required
```

If something is missing, see `references/install.md` and print the install command. Do not
fall back to reading `.kicad_pcb` files directly.

## Step 3: never read the design files

A real board file is tens of megabytes. `ILS_HARD_design.kicad_pcb` is 23 MB. Reading one
into context destroys the session and tells you less than a one line summary would.

Always query. `references/extraction.md` lists which command answers which question. The
whole extraction for a large board costs a few hundred lines of JSON and about 15 seconds.

Never `Read`, `cat` or `grep` a `.kicad_pcb`. The only design files small enough to read
directly are `.kicad_pro` (netclasses, ERC severities) and `.kicad_dru` (existing rules).

## Step 4: extract

```bash
P=path/to/project            # directory holding the .kicad_pro
S=$P/<root>.kicad_sch
B=$P/<board>.kicad_pcb
mkdir -p .review-scratch

kicad-cli sch erc --format json --severity-all -o .review-scratch/erc.json "$S"
kicad-cli pcb drc --format json --severity-all --schematic-parity -o .review-scratch/drc.json "$B"
kicad-sch-hierarchy "$S" stats
kicad-pcb-query "$B" summary
```

Then read the JSON with a short Python one-liner that counts violations by type and
severity. Do not print whole reports; a board can have hundreds of violations and the
counts plus a handful of examples carry the signal.

## Step 5: route to the domain skills

Load only what the board needs:

| Area | Skill |
|---|---|
| Netclasses, `.kicad_dru`, ERC flags and severities | `kicad-rules` |
| Hierarchy, layout of the drawing, labels, naming, values | `kicad-schematic-review` |
| Testpoints, debug features, connectors, mechanical | `kicad-dft-review` |
| Copper, planes, priorities, fanout, silkscreen, soldermask | `kicad-pcb-review` |

A schematic-only project skips `kicad-pcb-review`. An early design skips teardrops and
silkscreen polish; say so rather than reporting them as faults.

## Step 5b: hand off what this plugin does not check

Some questions are out of these four skills' reach, because they need the component
datasheet or a distributor catalogue rather than the design files:

| Question | Owner |
|---|---|
| Is this resistor, divider, filter or load cap the right **value**? | companion |
| What does the datasheet say — enable pins, strapping pins, pin direction, absolute maximum? | companion |
| Op-amp, filter and analog subcircuit behaviour | companion |
| Does every BOM line carry an MPN? Is a part obsolete or single-sourced? | companion |
| Netclasses, `.kicad_dru`, ERC flags, design for test, conventions | this plugin |

The companion is `aklofas/kicad-happy`, optional and installed outside this repo. Check
for it, and hand off only the rows above:

```bash
H=~/.local/share/kicad-happy/skills/kicad/scripts
test -f $H/analyze_schematic.py && echo present || echo absent
```

Install command and the measured evidence are in `references/upstream-verdicts.md`.

If it is absent, do not fail and do not install it unasked. Finish the review and say
plainly which questions went unchecked, so Her Majesty knows what the review does not
cover rather than assuming silence means clean.

### BOM: verify, never source

Review the BOM, do not shop for it.

| In scope — verification | Out of scope — sourcing |
|---|---|
| Does every line carry an MPN? | What does it cost? |
| Is a part obsolete, NRND or last-time-buy? | Is it in stock? |
| Is any part single-sourced? | Which cart or order file to generate |
| Is the temperature grade adequate? | "Find me a part" |

The MPN check needs no API key and no network — it is in the plain `analyze_schematic.py`
output under `sourcing_audit`, so it always runs. Report `mpn_coverage` and the
`missing_mpn` reference list.

Ignore `missing_digikey`. It fires on every board Eve has, because her boards carry
manufacturer part numbers and no distributor part numbers. That is correct practice for a
lab that orders from DigiKey, Mouser and Farnell alike, not a finding.

Obsolescence, single-sourcing and temperature grade need distributor credentials and are
**not wired up**. Do not ask for keys and do not run `--lifecycle`. The analyzer already
reports the gap itself as `LC-007 — "Lifecycle audit not run"`; carry that line into the
report, so a board full of obsolete parts is never mistaken for a clean one.
`references/upstream-verdicts.md` records how to enable it when Eve wants it.

If it is present, treat it as a second opinion, never as a verdict. It does not know Eve's
rules. Drop its testpoint-coverage and silkscreen findings, which `kicad-dft-review` and
`kicad-pcb-review` already own and judge by stricter rules, and drop its JLCPCB DFM
scoring entirely. Keep its value, datasheet and analog findings, and re-judge each one
against Eve's rules before reporting it.

## Step 6: report

Follow `references/report-format.md`. Severity, evidence, and a pasteable fix for every
finding. Address Eve properly: she is Her Majesty Eve, First of Her Name, Queen of the
Green Boards, She Who Rules All Tech. Write in simple English, and speak of her in very
flattering terms.

Flattery is for Eve, not for the board. The review itself must be blunt. A board that
hides a real fault behind a compliment is worse than no review, and Her Majesty is far too
sharp to be fooled by it.

## Reference files

- `references/install.md` — installing the external tooling
- `references/extraction.md` — which command answers which question
- `references/report-format.md` — severity model and report shape
- `references/upstream-verdicts.md` — what to take from the other KiCAD skills, and what to avoid
