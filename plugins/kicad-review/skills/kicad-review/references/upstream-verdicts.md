# The other KiCAD AI projects: what to take, what to leave

`EVE.md` lists four projects and asks what to copy from them. Verdicts below, checked in
October 2026.

Common bias to watch for in all of them: they are built to **create** boards, usually
cheap Chinese-manufactured ESP32-class boards, and their "review" is mostly manufacturing
readiness scoring. Eve wants review of engineering quality, for in-house instruments. Do
not inherit their definition of a good board.

## aklofas/kicad-happy — use it, as a second opinion

MIT, Python 3.10+, no required dependencies, around 1.5k stars. Eleven skills.

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
