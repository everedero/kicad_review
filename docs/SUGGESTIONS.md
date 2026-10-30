# Suggested changes to README.md and EVE.md

`AGENTS.md` says these two files must not be modified directly, only suggested. Nothing
below has been applied.

---

## 1. README.md — scope now contradicts EVE.md

`README.md` excludes "PCB design details", but `EVE.md` has a 50-line PCB design section
plus silkscreen, mechanical and soldermask rules, and Eve confirmed on 2026-10-28 that
PCB is in scope. Suggested replacement for the Scope and Exclusions sections:

```markdown
## Scope

* Schematics
* Design justification
* Rules (ERC and DRC)
* Netclasses
* Design for test and debug
* PCB design: copper, planes, routing priorities, fanout
* Silkscreen and soldermask

### Types of boards
* Interconnection boards
* Digital boards (MCU or digital ICs)

## Exclusions from scope

* Architecture
* Schematic and PCB *generation* (this project reviews, it does not create)
* Autorouting, component sourcing, SPICE, EMC simulation
```

Keep the excluded board types exactly as they are — analog, RF above 50 MHz, high voltage
above 35 V, and anything with batteries or chargers. The entry skill enforces that list.

---

## 2. EVE.md — the References section is empty

The four headings under "References" have no content. Verdicts, checked October 2026, are
written up in full in
`plugins/kicad-review/skills/kicad-review/references/upstream-verdicts.md`. Suggested
summary to paste under each heading:

```markdown
### aklofas/kicad-happy
MIT, pure Python, no required dependencies. **Use as a second opinion.** Take its
standalone scripts (analyze_schematic.py, analyze_pcb.py, cross_analysis.py,
diff_analysis.py), its separation of confidence from severity, and diff_analysis.py for
comparing board revisions. Leave its JLCPCB DFM scoring and its decoupling/ESD heuristics,
which do not know these rules.

### pann/kicad-aisch
A generator: design description to hierarchical schematic, LCSC-centric. **Nothing to take
for review.** Opposite direction of travel.

### Seeed-Studio/ai-skills
The schematic-analyzer skill. **Take the principle, not the code:** query the schematic and
return structured JSON, never dump raw files into context. Leave the pcbparts MCP
dependency and the Seeed parts catalogue.

### mixelpixx/Konnect
**Do not depend on it.** Archived October 2026 and AGPL-3.0. Its motivation was right —
escaping the Node + Python + wxPython + SWIG stack — and the lesson is to depend on
kicad-cli, which ships with KiCAD and cannot be abandoned.
```

---

## 3. EVE.md — Helper tools needs correcting

The current text says:

```
pip3 install kicad-tools
```

Three corrections:

1. The package is `rjwalters/kicad-tools`. Verified live: v0.24.0, released 2026-10-20,
   KiCAD 8/9/10, MIT, works offline with no running KiCAD. It is a good choice.
2. Prefer `uv tool install kicad-tools` over `pip3 install`, so it lands in its own
   environment with its 16 executables on `PATH`.
3. Its README is known to drift from the real CLI. Trust `--help`.

The most useful executables for review are worth naming in `EVE.md`, because they map
directly onto existing rules:

| Executable | Rule it serves |
|---|---|
| `kicad-sch-hierarchy` | "Always use hierarchical schematics", instantiated blocks |
| `kicad-pcb-stitch --dry-run --blanket --spacing 4.5` | stitching vias at 4–5 mm |
| `kicad-net-status --by-class` | netclass coverage |
| `kicad-pcb-query` | board summary, vias, traces, padmap for fanout |

### On "A spice tool?"

Left open in `EVE.md`. `kicad-happy` has a `spice` skill if it is wanted. Currently out of
scope for review.

---

## 4. EVE.md — a possible addition

Several rules are already KiCAD built-in checks and need switching on rather than
reviewing by hand. This fits the existing "setting up an ERC rule is preferred over
creating a skill" philosophy and may be worth stating outright:

| Rule in EVE.md | Built-in check |
|---|---|
| PCB antennae | `isolated_copper` |
| Thermal reliefs not in a big pour | `starved_thermal` |
| Silkscreen over pads | `silk_over_copper`, `silk_overlap` |
| Schematic and PCB disagree | `kicad-cli pcb drc --schematic-parity` |

Confirmed on the example boards: `isolated_copper` fires 8 times on `asynthosc`, finding
PCB antennae with no custom rule at all.
