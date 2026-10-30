# KiCAD review skills — design

Date: 2026-10-28

## Problem

`EVE.md` holds 211 lines of Her Majesty's schematic and PCB rules as prose. Prose does not
review boards. The public KiCAD AI skills that exist are built to *generate* boards, are
aimed at cheap ESP32-class designs, and define "good" as "manufacturable by JLCPCB".
Eve needs the opposite: review of engineering quality for in-house instruments.

## Approach

A Claude Code plugin of five skills that are thin routers over external tooling, plus the
rule assets that are the irreplaceable part of the method.

The organising principle comes from `EVE.md` itself:

> Setting up an ERC rule and flags to avoid an issue is preferred over creating a skill.
> Prefer making custom DRC rules rather than checking PCB manually.

Every finding that can be mechanised is delivered as pasteable `.kicad_dru` text, an ERC
severity change, or an ERC flag. Prose is the fallback, not the product. This keeps the
skills small and shifts the work into the project, where it keeps paying.

## Decisions

| Decision | Rationale |
|---|---|
| `EVE.md` overrides `README.md` on scope; PCB is in | Eve's call. `README.md` is stale and needs a suggested update. |
| No helper scripts in this repo | Eve's constraint. Use existing packages; reference upstream scripts via an install procedure. |
| Rule assets *do* live here | `.kicad_dru` and the flag symbols are declarative data, not scripts, and are the part nothing upstream provides. |
| Five skills, not one | Each maps to a contiguous `EVE.md` section, so a review loads only what it needs. Silkscreen folded into PCB review: eight lines does not justify a skill. |
| Skills never parse KiCAD files | `ILS_HARD_design.kicad_pcb` is 23 MB. Querying costs a few hundred lines of JSON and ~15 s. |
| `kicad-cli` is the authority | Ships with KiCAD, same engine as the GUI, honours the project's own `.kicad_dru`, cannot be abandoned. |

### Rejected dependencies

- `kicad-python`/`kipy` — needs a running GUI, so useless offline or in CI.
- `pcbnew` SWIG — deprecated in KiCAD 9, **removed in KiCAD 11**.
- `Konnect` — archived October 2026, AGPL-3.0.
- Any new S-expression parser — that is what the chosen tools are for.

## Architecture

```
.claude-plugin/marketplace.json
plugins/kicad-review/
  .claude-plugin/plugin.json
  skills/kicad-review/            entry: scope guard, doctor, extraction, routing, report
    references/{install,extraction,report-format,upstream-verdicts}.md
  skills/kicad-rules/             netclasses, .kicad_dru, ERC flags and severities
  skills/kicad-schematic-review/  hierarchy, conventions, naming, values
  skills/kicad-dft-review/        testpoints, debug features, connectors, mechanical
  skills/kicad-pcb-review/        copper, priorities, thermal, fanout, silkscreen, mask
  assets/dru/eve-netclasses.kicad_dru
  assets/dru/eve-geometry.kicad_dru
  assets/symbols/eve_flags.kicad_sym
```

### Tooling layers

1. `kicad-cli` 10.0.6 — ERC, DRC, schematic parity, netlist export.
2. `kicad-tools` (PyPI, MIT) — `kicad-sch-hierarchy`, `kicad-pcb-query`, `kicad-net-status`,
   `kicad-pcb-stitch`. Offline, JSON, no GUI.
3. `kicad-happy` (MIT, optional) — second opinion, `findings[]` JSON.
4. `jlcparts` — only for boards actually going to JLCPCB.

## Rule assets

`eve-netclasses.kicad_dru` encodes POWER 0.25 mm / HIGHPOWER 0.4 mm minimum widths,
MIDFREQ and HIGHFREQ clearances, a HIGHFREQ via budget, and constant-width rules using both
`(min)` and `(max)` so a *widened* section is caught as well as a necked one.

`eve-geometry.kicad_dru` covers edge clearance on all copper layers, via-to-pad clearance,
the GND-via-near-GND-pad check via the exclusion-then-assertion idiom, through-hole to
soldermask spacing for selective wave soldering, and testpoint spacing.

`eve_flags.kicad_sym` provides `PWR_IN_FLAG` (power_in), `INPUT_FLAG` (input) and
`OUTPUT_FLAG` (output). All three are power symbols with pin name `~`, so like KiCAD's own
`PWR_FLAG` they attach without renaming the net.

Two limits are stated in the assets rather than hidden: the clearance rules hard-code 3x an
assumed netclass width because DRC cannot read local track width, and the guard ring cannot
be proven by DRC at all because it is a topology claim.

## Traceability

Every `EVE.md` section maps to at least one skill.

| `EVE.md` section | Skill |
|---|---|
| Generalities (simple English, forms of address) | `kicad-review` → `references/report-format.md` |
| References (the four upstream projects) | `kicad-review` → `references/upstream-verdicts.md` |
| Helper tools | `kicad-review` → `references/install.md` |
| Schematics — netclasses and DRC | `kicad-rules` |
| Schematics — hierarchy, drawing, naming, values | `kicad-schematic-review` |
| Schematics — PWR_FLAG, PWR_IN_FLAG, INPUT/OUTPUT flags, ERC | `kicad-rules` |
| Testpoint rules and guidelines | `kicad-dft-review` |
| Mechanical | `kicad-dft-review` |
| Debug and measure features | `kicad-dft-review` |
| PCB design | `kicad-pcb-review` |
| Silkscreen | `kicad-pcb-review` |
| Others (soldermask colour) | `kicad-pcb-review` |

### Known gaps, recorded rather than dropped

- **Guard ring presence** — not provable by DRC; needs a plot and human judgement.
- **Drawing conventions** (GND down, pull-ups vertical, inputs left) — needs the rendered
  sheet; no tool extracts intent from coordinates.
- **Testpoints not moving between revisions** — needs two revisions to compare.
  `kicad-happy`'s `diff_analysis.py` is the candidate tool.
- **Reversed-connector analysis** — inherently a judgement call about what is on the pins.
- **SPICE** — `EVE.md` asks "A spice tool?" and leaves it open. Out of scope here;
  `kicad-happy` has one if it is wanted later.

## Verification

Run against the three real boards in `exemples/`, from three designers.

Tooling, measured:

| Board | Size | Footprints | Nets | ERC | DRC + parity |
|---|---|---|---|---|---|
| `asynthosc` | 4.2 MB | 268 | 210 | 23 violations, ~2 s | 38 violations, 4.8 s |
| `ODY` | 4.0 MB | 321 | 152 | — | — |
| `ILS_HARD_design` | 23 MB | 775 | 876 | — | 236 violations + 187 parity, 12.9 s |

Rule assets: both `.kicad_dru` files parse under `kicad-cli pcb drc` with zero rule-syntax
violations; all three symbols parse and plot under `kicad-cli sym export svg`.

Known-truth findings, all confirmed:

- `asynthosc` and `ODY` have only a `Default` netclass → POWER/HIGHPOWER missing.
- `ILS_HARD_design` has ten per-*net* classes (`P3V3_OUT_A`, `M12V_VOX`, …) and none of
  Eve's four functional classes.
- `ILS_HARD_design` has 17 `TP*` footprints for 775 footprints and 876 nets, about 2%,
  against `ODY`'s 126 for 321. This corroborates `review_ioda/ioda_absence_de_tp.png`.
- `isolated_copper` fires 8 times on `asynthosc` — those are PCB antennae, found by a
  built-in check rather than by eye.

## Out of scope

Generation, autorouting, sourcing, SPICE, EMC simulation. Upstream covers these; this
repo reviews.
