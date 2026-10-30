Do not modify README.md or EVE.md directly.
You can suggest modifications.

EVE.md is the human input for the project, it gives the base concepts to be deployed as skills.
Suggested changes to those two files are collected in `docs/SUGGESTIONS.md`.

## What this repo is

A Claude Code plugin of KiCAD **review** skills built from `EVE.md`. It reviews designs. It
never generates schematics, never autoroutes, and never edits the design under review.

```
.claude-plugin/marketplace.json   plugin marketplace entry
plugins/kicad-review/
  skills/kicad-review/            entry point: scope guard, tooling doctor, routing
  skills/kicad-rules/             netclasses, .kicad_dru, ERC flags  (the core)
  skills/kicad-schematic-review/  hierarchy, conventions, naming, values
  skills/kicad-dft-review/        testpoints, debug features, connectors
  skills/kicad-pcb-review/        copper, routing, thermal, silkscreen, soldermask
  assets/dru/*.kicad_dru          rule templates, validated against kicad-cli
  assets/symbols/eve_flags.kicad_sym  PWR_IN_FLAG, INPUT_FLAG, OUTPUT_FLAG
docs/superpowers/specs/           design spec
exemples/                         three real boards for validation (gitignored)
```

## Conventions

- **No helper scripts in this repo.** Use `kicad-cli` and `kicad-tools`; reference upstream
  scripts by path. The only things shipped here are declarative rule assets.
- **Never read a `.kicad_pcb`.** One of the example boards is 23 MB. Query instead; see
  `plugins/kicad-review/skills/kicad-review/references/extraction.md`.
- **Prefer a rule over a report.** A finding that can be a `.kicad_dru` rule, an ERC
  severity or an ERC flag should be delivered as that, with text to paste.
- Code and comments in English.

## Commands

```bash
# Tooling (kicad-cli ships with KiCAD; verified against 10.0.6)
uv tool install kicad-tools

# Validate a rule asset actually parses — do this after editing any .kicad_dru
mkdir -p .review-scratch/dru-test
cp exemples/asynth2osc/kicad/asynthosc/asynthosc.kicad_{pcb,pro} .review-scratch/dru-test/
cp plugins/kicad-review/assets/dru/eve-geometry.kicad_dru .review-scratch/dru-test/asynthosc.kicad_dru
kicad-cli pcb drc --format json -o /tmp/t.json .review-scratch/dru-test/asynthosc.kicad_pcb

# Validate the symbol library parses
kicad-cli sym export svg -o /tmp/symsvg plugins/kicad-review/assets/symbols/eve_flags.kicad_sym
```

A `.kicad_dru` that KiCAD rejects is worse than no rule: the project looks protected and is
not. Always validate after editing.

## Validation boards

`exemples/` holds three real boards from three designers, with real flaws. They are nested
git repositories and are gitignored. Known ground truth, useful as regression targets:

- `asynth2osc` (268 footprints) and `digital_output_input_interface/ODY` (321): only a
  `Default` netclass.
- `ioda_mother_board/ILS_HARD_design` (775 footprints, 876 nets, 23 MB): ten per-*net*
  netclasses instead of functional ones, and only 17 testpoints.
- `asynth2osc` is an analog synth board, so the scope guard must fire on it.
