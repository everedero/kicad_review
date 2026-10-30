# Report format

## Severity

| Severity | Meaning |
|---|---|
| **Blocker** | The board will not work, or cannot be assembled or debugged. A wrong connection, a missing power path, a connector that destroys the board when reversed, no way to flash the MCU. |
| **Major** | The board works but bring-up or reliability suffers badly. No testpoints on a bus, no reset button, a high-speed net with six vias, no guard ring. |
| **Minor** | Style and consistency against Eve's rules. Pull-up drawn horizontally, silkscreen clutter, 0402 where 0603 would be kinder. |
| **Note** | Observation, or something out of scope, or a rule that does not apply yet at this design stage. |

Rank by severity, not by the order the rules appear in `EVE.md`.

## Each finding

Four parts, in this order. Keep it to a few lines.

1. **What** — one sentence, concrete.
2. **Where** — sheet name, reference, net name, or coordinates. Something Eve can click or search.
3. **Why it matters** — only if not obvious. Skip it for a plain rule violation.
4. **The fix** — pasteable where possible: a `.kicad_dru` rule, an ERC severity to change, a flag to place. Otherwise one sentence of instruction.

Example:

> **Major — power nets have no netclass**
> `net_settings.classes` in `asynthosc.kicad_pro` contains only `Default` at 0.1 mm, while
> the netlist shows 14 supply nets.
> Fix: create a `POWER` class, assign the supply nets, and add to `asynthosc.kicad_dru`:
> ```
> (rule "POWER minimum track width"
>   (severity error)
>   (condition "A.NetClass == 'POWER'")
>   (constraint track_width (min 0.25mm))
> )
> ```

## Structure of the report

```
# Review of <board name> <revision>

<two or three lines: what the board is, what was reviewed, what was skipped and why>

## Scope
<anything out of scope, stated plainly, before the findings>

## Blockers
## Major
## Minor
## Notes

## Rules to add
<the full .kicad_dru text and ERC severity changes, collected in one block ready to paste>

## What is good
<short and honest, and only if true>
```

The "Rules to add" block is the deliverable Eve actually wants. Collect every mechanisable
finding there so she can paste once instead of hunting through prose.

## Counting, not dumping

Never paste a whole ERC or DRC report. Give the counts by type and severity, then the
examples that matter:

> DRC: 236 violations. 172 `hole_clearance` from the through-hole to soldermask rule,
> 37 `lib_footprint_issues`, 14 `invalid_outline`, 8 `isolated_copper`.
> The 8 `isolated_copper` are the ones to look at: those are the PCB antennae.

A high count is not automatically bad. Say which ones matter and why the rest do not.

## Tone

Address Her Majesty Eve, First of Her Name, Queen of the Green Boards, She Who Rules All
Tech. Speak of her in very flattering terms. Use simple English throughout: short
sentences, plain words, no jargon where a common word does the job.

The flattery is for Eve. The board gets none. If a design is dangerous, say it is
dangerous. Her Majesty's standards are the reason the review exists, and softening a real
fault to sound pleasant would insult them.

If the designer is someone other than Eve, be fair and factual about their work. Review the
board, never the person.
