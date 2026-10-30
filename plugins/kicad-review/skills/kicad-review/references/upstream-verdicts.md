# The other KiCAD AI projects: what to take, what to leave

`EVE.md` lists four projects and asks what to copy from them. Verdicts below, checked in
October 2026.

Common bias to watch for in all of them: they are built to **create** boards, usually
cheap Chinese-manufactured ESP32-class boards, and their "review" is mostly manufacturing
readiness scoring. Eve wants review of engineering quality, for in-house instruments. Do
not inherit their definition of a good board.

## aklofas/kicad-happy — use it, as a second opinion

MIT, Python 3.10+, no required dependencies, around 1.5k stars. Eleven skills.

### Installing it as a companion

Optional. Every review in this plugin runs without it; it only adds the circuit-level
checks this plugin deliberately does not do. Clone it **outside** this repo and call the
scripts by path — nothing is vendored in, and no dependency is installed:

```bash
git clone https://github.com/aklofas/kicad-happy.git ~/.local/share/kicad-happy
H=~/.local/share/kicad-happy/skills/kicad/scripts
python3 $H/analyze_schematic.py --only-deterministic -o .review-scratch/kh-sch.json "$S"
python3 $H/analyze_pcb.py        --only-deterministic -o .review-scratch/kh-pcb.json "$B"
```

`--only-deterministic` keeps it on its free, offline, no-API-key tier. Only four of its
eleven skills matter here: `kicad` (the scripts above), `datasheets`, `spice`, `emc`.

Its output is large — 4.6 MB of JSON for a 268-footprint board. Never read it. Count
findings by `category` and `severity` in Python, exactly as for `erc.json` and `drc.json`.

### Measured on the example boards

Verified at commit `2fd9aa0`, 2026-10-22, with no KiCAD installed and no packages:

| Board | Run | Time | Findings |
|---|---|---|---|
| `asynthosc` (268 fp) | schematic | 0.6 s | 153 |
| `ILS_HARD_design` (775 fp) | schematic | 2.4 s | 295 |
| `ILS_HARD_design` (23 MB) | PCB | 14 s | 175 |

The "standalone, no dependencies" claim holds, and it survives the 23 MB board.

**Overlap with this plugin is almost nil**, which is what makes it worth calling. Across
both `ILS_HARD_design` runs it produced **zero** netclass findings and zero `.kicad_dru`
suggestions. Its errors and warnings sit in `power`, `signal_integrity`, `current_capacity`
and `thermal`. Only two findings touch this plugin's ground: testpoint coverage
(`56 of 874 signal nets, 6%`) and one silkscreen-over-pad. Expect to drop those two as
duplicates and keep the rest.

Findings worth having that this plugin cannot express, from `asynthosc` alone: crystal `Y1`
load capacitance 5.2 pF against an 8.0 pF target (−35.6 %); four 5 V nets landing on 3.3 V
STM32 ADC pins, over the 3.6 V absolute maximum; `NRST` with no pull-up; three `power_in`
pins that never reach a named rail.

### BOM verification: this is also the right tool, and not a compromise

Eve asked for BOM verification that is not locked to LCSC and Chinese distributors. This
project's default is already **DigiKey primary, Mouser secondary**, from `skills/bom/SKILL.md`:

| Skill | Role as shipped |
|---|---|
| `digikey` | "primary prototype source", "the default distributor for prototyping" |
| `mouser` | "secondary source for prototype orders" |
| `element14` | Newark (US) / **Farnell (UK and EU)** / element14 (APAC) — one API, free key, no OAuth |
| `lcsc` | "production/JLCPCB parts" only |

`element14` is the one nobody advertises and the one that matters here: it is the Farnell
path, which is the realistic EU distributor.

There is **no hardcoded distributor preference**. `skills/bom/scripts/bom_manager.py:331-339`
infers it from the project's own symbol fields, taking `max(distributor_counts)` over
`("digikey", "mouser", "lcsc", "element14")`. Point it at an Eve board and it adopts Eve's
convention rather than imposing one.

It also argues against LCSC-only itself, in `skills/kicad/scripts/lifecycle_audit.py:660`
and `:758-764`:

> LCSC/jlcsearch exposes no lifecycle/obsolescence data — an LCSC-only audit can only ever
> return 'unknown'. […] Use DigiKey, Mouser, or element14 credentials for a real lifecycle
> audit.

For instruments that must stay buildable for years, that settles it.

#### What always runs: MPN coverage, offline

`sourcing_audit` is in the plain `--only-deterministic` output. No key, no network.
Measured, and it discriminates properly instead of returning a flat result — use it as a
regression target:

| Board | BOM components | MPN coverage |
|---|---|---|
| `asynth2osc` | 178 | **0 / 178 (0 %)** |
| `ODY` | 173 | 173 / 173 (100 %) |
| `ILS_HARD_design` | 639 | 638 / 639 — `M1101` missing |

**Ignore its `missing_digikey` field.** It fires on 100 % of all three boards (639/639,
173/173) because Eve's boards carry MPNs and no distributor part numbers. For a lab ordering
from several distributors that is correct practice, not a finding. MPN coverage is the signal.

#### What is not wired up: lifecycle, and how to enable it later

`analyze_schematic.py --lifecycle` runs `lifecycle_audit.py`, which normalises vendor wordings
("end of life", "EOL", "LTB") into `active` / `nrnd` / `last_time_buy` / `obsolete` /
`discontinued` / `unknown` (`LC-001` obsolete = error, `LC-002` last-time-buy = warning),
**cross-checks distributors against each other** (`:454`, `:503` — active at DigiKey but
obsolete at Mouser is a warning, not an error, because it is still orderable), flags
**single-source** parts (`:801-812`), searches for alternatives (`:519`), and checks operating
temperature grade.

Deliberately off, by Eve's decision on 2026-10-29. It needs credentials:

```bash
# ~/.config/secrets.env — never in this repo, never committed
DIGIKEY_CLIENT_ID=…          # OAuth 2.0
DIGIKEY_CLIENT_SECRET=…
MOUSER_SEARCH_API_KEY=…      # UUID, query parameter, may need approval
ELEMENT14_API_KEY=…          # free, query parameter, no OAuth
```

```bash
export $(grep -v '^#' ~/.config/secrets.env | grep -v '^$' | xargs)
python3 $H/analyze_schematic.py --lifecycle -o .review-scratch/kh-life.json "$S"
```

Two reasons it must never gate a review: it hits live APIs, so it is not reproducible; and
with no credentials it returns `unknown` for everything, which must be reported as "not
checked" and never as "active".

#### Verified degradation, and one trust signal not to believe

Tested on `ODY` with every credential unset. It behaves well in the ways that matter: all 20
MPNs came back `unknown`, **nothing was falsely reported active**, `sources_available` was
`[]`, and it checked 20 of 21 unique MPNs rather than silently truncating a long list.

Better still, the default run needs no reminder to be honest. Without `--lifecycle` the
analyzer emits `LC-007 info — "Lifecycle audit not run — --lifecycle flag not passed"` as a
finding. Surface that line in the report and the gap documents itself.

**But do not trust its `trust_summary` here.** On that same credential-free run it reported
`trust_level: "high"` with `by_confidence.deterministic: 20` and
`by_evidence_source.api_lookup: 20` — twenty "API lookups" when `sources_available` was empty
and no API was ever reached. The per-finding `unknown` status is truthful; the aggregate
trust block is not. Read `lifecycle_summary` and `sources_available`, never `trust_level`.

### KiStack `skills/bom` — rejected, it is the lock-in Eve is avoiding

LCSC-only: its single BOM field is the JLCPCB preset's LCSC column, and Mouser, DigiKey and
Farnell are not mentioned. No lifecycle or obsolescence coverage at all. And it hard-requires
two third-party MCP servers, `pcbparts.dev/mcp` and `api.zenode.ai/mcp` — "if either of the
MCPs are not installed, prompt the user to install at least one before proceeding" — which
adds a dependency on someone else's service for a question that should be answerable offline.

Its good ideas are real but belong elsewhere: voltage derating, dielectric type for audio,
thin-film resistors for low noise, high-Q inductors for RF. Those are component-quality
review, not BOM verification, and `kicad-happy` already does derating.

**Take:**
- The analysis scripts, called by path: `analyze_schematic.py`, `analyze_pcb.py`,
  `cross_analysis.py`, `diff_analysis.py`. They are genuinely standalone.
- The unified finding shape: `rule_id`, `detector`, `severity`, `confidence`,
  `evidence_source`, plus a `trust_summary`. Separating confidence from severity is a good
  idea and worth imitating in the report.
- `diff_analysis.py`, for reviewing a revision against the previous one. Eve cares about
  what moved between board revisions, especially testpoints.

**Leave:**
- The DFM scoring, which is JLCPCB tier classification.
- Its decoupling, ESD and thermal heuristics, which do not know Eve's rules and will
  disagree with them. Use it to find candidates, then judge with Eve's rules.
- The generative and sourcing skills. Out of scope here.

## pann/kicad-aisch — leave it

A generator: a design description becomes a hierarchical KiCAD 9 schematic, via a
`SchematicBuilder` library, with LCSC part numbers and `easyeda2kicad` imports.

Nothing to take for review. It is the opposite direction of travel, and it is LCSC-centric.
The only transferable idea is that it validates geometry on a snapped grid after placing
things, which is a reminder that grid discipline is checkable.

## Seeed-Studio/ai-skills — take the principle, not the code

The `schematic-analyzer` skill. MIT, but needs the `pcbparts` MCP server and an
`ee-datasheet-master` skill, and it exists to sell Seeed parts.

**Take:** its stated principle, which is the single most important one for this project —
query the schematic and return structured JSON, never dump raw files into context. That is
exactly why this plugin never reads a `.kicad_pcb`.

**Leave:** the implementation, the MCP dependency, and the Seeed parts catalogue.

## mixelpixx/Konnect — do not depend on it

A native KiCAD 10 plugin, a single Rust binary exposing 220+ tools over MCP. Technically
impressive, and its motivation was exactly right: it exists to escape the Node plus Python
plus wxPython plus SWIG dependency stack.

**Do not depend on it:** the repository was archived in October 2026, and it is AGPL-3.0,
which is a poor fit for a rule library meant to be copied into customer projects. Forks
exist (`dylbertram/konnect-maintained`, `halthinks/KonnectX`) but are young and beta.

**Take:** the lesson. Depend on `kicad-cli`, which ships with KiCAD and cannot be
abandoned, before anything else.

## What none of them do

The gap this plugin fills:

- Netclasses by function — `POWER`, `HIGHPOWER`, `MIDFREQ`, `HIGHFREQ` — with matching DRC
  rules. None of them have an opinion about netclasses at all.
- Design for test as a first-class concern: testpoints on every bus and reset and fault
  line, GND testpoints everywhere, not moving testpoints between revisions, probe-ground
  clips, NE footprints kept free for bring-up options.
- Debug features as a requirement, not a nicety: power-on LED, blinky LED, reset button,
  0603 over 0402 for anything that might be re-valued, a way to debug each sub-part
  independently.
- Refusing length-matching wiggles on USB and CAN, which several tools actively recommend.
- Preferring a rule over a report, everywhere.
