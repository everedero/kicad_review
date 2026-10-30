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

---

## 5. EVE.md — three gaps nobody in the ecosystem covers

A survey of the third-party KiCAD agent projects on 2026-10-29 settled where this plugin
stops and `kicad-happy` starts (now wired up in Step 5b of the entry skill). It also found
three things **no project covers at all**. They are candidates for original skills here,
and may be worth naming in `EVE.md` as intent:

**(a) Conformance to a manufacturer reference design.** Nothing checks a layout against a
reference design. The whole ecosystem offers only visual Gerber area-diff — OpenGerberDiff,
`pcb-diff`, `gerbv` XOR mode — which answers "what changed between revision A and B", not
"does my board follow TI's layout guidance". `kicad-happy` gets halfway: it reads a
manufacturer's PDF schematic to extract subcircuits and check values against the datasheet.
The missing piece fits this project's philosophy exactly: turn the layout section of a
datasheet or application note into a checklist, then answer each item with a `kicad-cli` or
`kicad-pcb-query` query. Prefer a rule over a report, applied to app notes.

**(b) Configuration and strapping pins, and pin direction, as a first-class review.** This
is the one place this plugin is already ahead of the field. `eve_flags.kicad_sym` ships
`INPUT_FLAG` and `OUTPUT_FLAG`; `kicad-happy` only classifies pin electrical type inside a
generic pinout extraction, and buries enable and strapping pins there with no dedicated
check. Boot straps and mode pins silently pulled the wrong way are a classic bring-up loss,
and an ERC flag can catch them. Worth growing rather than delegating.

**(c) Analog review rules of Eve's own.** Currently delegated to the companion, and the
scope guard fires on analog boards. Fine as a position — but it is a position, not a gap in
the tooling, and `asynth2osc` is a real board that gets only a pointer today.

---

## 6. One asset worth borrowing, two worth reading

Not skills — none of these should be depended on, but each solves a problem this project
has already met.

**`xtopher88/pcb_tool_pdf_extract`** — its vendor section-heading vocabulary is the single
most reusable thing found in the survey. One list matching the different wordings vendors
use for the same datasheet section: ST's "Pin description", TI's "Pin Configuration and
Functions" and "Terminal Functions", "Signal Descriptions". Any future datasheet work here,
including gap (a) and (b) above, needs exactly that list. Worth copying as a reference file,
with attribution.

**`Gato513/kicad-mcp`** — encodes board state as TOON, a compact format chosen to keep token
use low across many small tool calls. Prior art for the same problem that gave this repo its
"never read a `.kicad_pcb`" rule. Worth reading before extending `references/extraction.md`.
The project itself is write-capable and autoroutes, so it stays out.

**The commercial framing**, for context only: Circuit Mind's Agentic Verification Engine
splits the job across agents — some read datasheets for pin configuration and voltage
limits, others trace nets and buses, then a separate QA agent checks the findings before
reporting. The useful idea is the last one: a review pass over the review.

---

## 7. EVE.md and README.md — BOM verification is now in scope, and neither file says so

`EVE.md` mentions BOM, sourcing, distributors and obsolescence **nowhere** — zero hits. Eve
asked for BOM verification on 2026-10-29 and settled the scope as *verification in, sourcing
out*, now implemented in Step 5b of the entry skill. Both files need to catch up.

### README.md — the exclusion as worded is too broad

Suggestion 1 above proposes an exclusions list containing "component sourcing". That wording
excludes BOM verification too, which is no longer intended. Suggested replacement for that
one line:

```markdown
* Component sourcing: pricing, stock, carts and order files
  (BOM *verification* — MPN coverage, obsolescence, single-sourcing — is in scope)
```

### EVE.md — suggested new section

```markdown
## BOM

Review the BOM, do not shop for it.

* Every BOM line must carry a manufacturer part number. A distributor part number is not a
  substitute: we order from DigiKey, Mouser and Farnell, so the MPN is the real identity.
* No obsolete, NRND or last-time-buy parts in a new design.
* Flag any part with a single active source. An instrument that must stay buildable for
  years cannot depend on one distributor's stock.
* Not in scope: price comparison, stock levels, order files, "find me a part".
* Not LCSC-only, ever: LCSC publishes no lifecycle data, so an LCSC-only check cannot
  answer the obsolescence question at all.
```

Confirmed on the example boards, offline and with no API key: `asynth2osc` carries **0 MPNs
out of 178 components**, while `ODY` is at 173/173 and `ILS_HARD_design` at 638/639 (only
`M1101` missing). So the rule has teeth and one board already fails it outright.

---

## 8. For later — the one trap worth writing down

The companion's `missing_digikey` metric fires on 100 % of all three example boards (639/639
and 173/173) because Eve's boards carry MPNs and no distributor part numbers. That is correct
practice, not a defect. Any future BOM rule must count MPN coverage and ignore distributor
part-number coverage, or it will report every board as broken.
