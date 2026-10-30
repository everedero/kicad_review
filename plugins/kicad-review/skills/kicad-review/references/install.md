# Installing the review tooling

No helper scripts live in this repository. Everything below is external and installed
once. The repository ships only the rule assets under `assets/`.

## Required

### 1. kicad-cli

Comes with KiCAD 9.0 or later. This is the authority: it is the same engine the GUI uses,
it honours the project's own `.kicad_dru`, and it is the only tool whose ERC and DRC
verdicts can be quoted as fact.

```bash
kicad-cli --version      # verified against 10.0.6
```

Install with KiCAD itself. On Debian and Ubuntu it is in the `kicad` package.

### 2. kicad-tools

MIT, pure Python, works offline with no running KiCAD instance.
Source: <https://github.com/rjwalters/kicad-tools> (PyPI `kicad-tools`).

```bash
uv tool install kicad-tools     # preferred
# or: pipx install kicad-tools
```

Installs `kct` plus focused executables, of which these matter for review:

| Executable | Use |
|---|---|
| `kicad-sch-hierarchy` | sheet tree, depth, instance count, label and pin consistency |
| `kicad-pcb-query` | board summary, footprints, nets, traces, vias, stackup |
| `kicad-net-status` | connectivity per net, `--by-class` for netclass coverage |
| `kicad-pcb-stitch` | stitching via analysis, `--dry-run` to report without touching |
| `kicad-symbols`, `kicad-nets`, `kicad-bom` | schematic side queries |

Its README is known to drift from the real CLI. Trust `--help`, not the README.

If `kct` is not on `PATH` after install, add `~/.local/bin`.

## Optional

### 3. kicad-happy

MIT, zero required dependencies, Python 3.10+.
Source: <https://github.com/aklofas/kicad-happy>

Useful for its second opinion: `analyze_schematic.py`, `analyze_pcb.py` and
`cross_analysis.py` emit a unified `findings[]` JSON with severity and evidence. Clone it
outside this repository and call the scripts by path:

```bash
git clone https://github.com/aklofas/kicad-happy ~/.local/share/kicad-happy
python3 ~/.local/share/kicad-happy/skills/kicad/scripts/analyze_pcb.py board.kicad_pcb \
    --analysis-dir .review-scratch/ --full
```

Treat its output as a hint, not a verdict. Its DFM scoring is JLCPCB-shaped and its
decoupling and ESD heuristics do not know Eve's rules.

### 4. jlcparts

Only for boards actually going to JLCPCB, as noted in `EVE.md`.
Source: <https://github.com/yaqwsx/jlcparts>

## Deliberately not used

| Tool | Why not |
|---|---|
| `kicad-python` / `kipy` | Needs a running KiCAD GUI with the API server enabled. Useless for offline or CI review. |
| `pcbnew` SWIG bindings | Deprecated in KiCAD 9, removed in KiCAD 11. Anything built on it dies at the next upgrade. |
| `Konnect` | Archived October 2026, and AGPL-3.0, which is a poor fit for a rule library. |
| Hand-written S-expression parsers | The whole point of the tools above. Do not write another one. |
