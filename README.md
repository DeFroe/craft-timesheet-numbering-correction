# craft-timesheet-numbering-correction

A [Claude Code](https://claude.com/claude-code) skill for craft/trade and service businesses
(construction, landscaping, cleaning, any subcontractor-based trade) that:

1. **Numbers documents** — reads a per-project tracking spreadsheet and works out the next free
   running document number, optionally in two independent sequences (a primary number used on
   documents, and a secondary number for internal-only counting, e.g. per subcontractor).
2. **Reviews handwritten timesheets** — takes a photographed/scanned timesheet, computes the
   correct worked hours under a configurable break-time rule, compares it against what was
   actually written down, produces a red-annotated correction copy where they disagree, and
   generates a clean digital timesheet from a template.

Nothing about a specific company, folder layout, or spreadsheet is hardcoded — a one-time setup
interview on first use captures your own folder path, terminology, break-time rule and template
layout into a config file.

## Install

Clone this repository straight into your Claude Code skills folder:

```
git clone https://github.com/DeFroe/craft-timesheet-numbering-correction.git \
  ~/.claude/skills/craft-timesheet-numbering-correction
```

Alternatively, download the repository and copy its contents (the files alongside `SKILL.md`)
into `~/.claude/skills/craft-timesheet-numbering-correction/`.

## First use

The first time you ask the skill to do anything, it notices there's no configuration yet and
walks you through a short setup interview: your base project folder, how your tracking file is
named, labels for the two parties on a timesheet, whether you need a second internal-only
numbering sequence, your break-time rule (or the bundled example), and which timesheet template
to use (the bundled example, or your own spreadsheet — the skill will look at your file and
propose a cell mapping for you to confirm).

Your answers are saved to:

```
~/.claude/skill-config/craft-timesheet-numbering-correction/config.json
```

— outside the skill folder itself, so updating the skill later never touches your configuration.
Say "reset setup" any time to redo the interview.

See [`references/setup.md`](references/setup.md) for the full interview script and config schema,
and [`config.example.json`](config.example.json) for a filled-out example.

## What's included

```
SKILL.md                            entry point: loads config, routes to the two workflows below
config.example.json                 documented example configuration
references/
  setup.md                          first-use interview + config schema
  numbering.md                      full numbering workflow
  timesheet-processing.md           full timesheet review/correction workflow
assets/
  tracking-template.xlsx            bundled example project-tracking spreadsheet
  timesheet-template.xlsx           bundled example timesheet spreadsheet
```

## Notable implementation details

A few things this skill's workflow gets right on purpose, documented in
`references/timesheet-processing.md`:

- **Exact digit position via pixel analysis, not guesswork.** Photos of paper forms are often
  slightly perspective-distorted; the workflow locates the actual ink position of a handwritten
  digit by scanning pixel brightness in a narrow window, rather than estimating row height.
- **Embedded fonts for PDF annotations.** Inserting red correction text with a non-embedded
  base-14 PDF font (e.g. plain "Helvetica") can look fine in the tool that created it, but get
  mis-decoded into garbled characters by a different PDF tool downstream (for example when a
  separate invoicing tool merges the file later). The workflow always embeds a real font file
  instead.
- **Exclusive break-rule boundaries.** A break-time rule table with tiers like "up to 6h → no
  break, up to 9h → 30 min" needs its boundaries treated as *exclusive* upper bounds — otherwise a
  value of exactly 9.00 lands in the wrong tier. This is called out explicitly wherever the rule
  is applied or turned into a spreadsheet formula.
- **Two independent numbering sequences.** A document-facing number and a purely-internal
  tracking number are easy to conflate; the workflow treats them as clearly separate concerns
  with separate labels.

## Limitations

- **No Excel COM automation in a sandboxed shell.** Converting the generated `.xlsx` timesheet to
  PDF via Excel automation (`pywin32`/COM) can fail in a sandboxed or restricted-token shell
  environment on Windows, even with Excel correctly installed — this is a hard boundary of that
  kind of environment, not a fixable setting. The skill detects this and asks you to export the
  PDF yourself once (File → Export → Create PDF/XPS) rather than silently failing or retrying.
- **Handwriting recognition needs a human check before billing.** This skill reads handwritten
  timesheets via image analysis; always review the generated correction copy and clean timesheet
  before using them for real invoicing or payroll.
- Tested/developed on Windows; the numbering and image-based workflows themselves aren't
  platform-specific, but the PDF-export fallback advice above is Windows/Excel-specific.

## License

MIT — see [LICENSE](LICENSE).
