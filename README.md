# Craft Timesheet Numbering & Correction

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) for craft and trade businesses that work with subcontractors: construction, landscaping, cleaning, building services. Two jobs come up constantly in that kind of shop, and always in the same order: work out which running document number is next for a project, then turn a photographed, handwritten timesheet into something you can actually bill from. The second part is where the tedium lives - recomputing every row, applying the break rule, finding the rows where the written total quietly disagrees with the arithmetic, and marking the correction so the subcontractor can follow what changed and why.

It works across any number of projects. Nothing about a specific company, folder layout or spreadsheet is baked in: a one-time setup interview on first use captures your own paths, terminology, break-time rule and template layout into a single config file.

## Install

Clone this repo directly into your Claude Code skills folder:

```bash
git clone https://github.com/DeFroe/craft-timesheet-numbering-correction ~/.claude/skills/craft-timesheet-numbering-correction
```

Or download it and copy the folder contents into `~/.claude/skills/craft-timesheet-numbering-correction/` yourself. Either way, restart/reopen Claude Code so it picks up the new skill.

## Quick start

1. Ask for something the skill covers - e.g. "what's the next document number for the Oakfield job?" or "there's a new timesheet photo in the Oakfield folder, check it".
2. On first use there's no config yet, so the skill runs a short setup interview: your base project folder, how tracking files are named, what you call the two parties on a timesheet, whether you need a second internal-only number, your break-time rule, and which timesheet template to use. Pick the bundled example template or your own spreadsheet - if it's your own, the skill reads it and proposes a cell mapping for you to confirm rather than guessing coordinates.
3. Your answers are saved to `~/.claude/skill-config/craft-timesheet-numbering-correction/config.json`, deliberately outside the skill folder, so updating the skill later can never overwrite your configuration. Say "reset setup" any time to redo the interview.

Asking only for a number is read-only: it won't write to your tracking file and won't create a document. Writing happens only when you ask for a timesheet to actually be processed.

## What a run looks like

An illustrative walkthrough rather than a captured transcript, but it's the shape every run takes. A subcontractor has dropped off a timesheet for the Oakfield job and you've photographed it into the project folder.

**Step 1 - locate the project and the number.** The skill finds the project folder, reads its tracking file, and reports what it found before touching anything:

> Project "Oakfield Sports Hall", client Meridian Estates, subcontractor Vance Groundworks. Last document number used is 014, so this one becomes 015. Vance Groundworks is on their 6th original for this project, which is the internal count only and won't appear on the document.

**Step 2 - recompute the hours.** Each row is read off the photo and recalculated from scratch, then checked against what was written down:

> Row 3 reads 7:00 to 16:00, so 9.00 hours worked. Your break rule puts anything from 9.00 hours upward at 0.75 h, and the sheet has 0.50 h. Net hours should be 8.25, not 8.50. The other four rows agree with the arithmetic.

That row is exactly the case the break rule is easiest to get wrong on, because 9.00 sits precisely on a tier boundary. The rule treats boundaries as exclusive upper bounds, so 9.00 belongs to the tier above, not below.

**Step 3 - mark the correction.** A copy of the original is annotated in red: the correct value in square brackets next to the wrong one, and the wrong one struck through. The original photo itself is never overwritten. Rather than estimating where row 3 sits vertically, the skill finds the actual ink position of the digit by scanning pixel brightness, because a photo of a paper form is rarely perfectly square to the camera.

**Step 4 - produce the clean timesheet.** The template is filled in with the corrected figures under document number 015: addresses, project, date, one row per work entry, break and net-hours formulas, both summary totals. One row per entry stays one row per entry - nothing gets condensed into a summary line.

**Step 5 - record it.** Number 015 is written into the tracking file with its date, and the internal count for Vance Groundworks moves to 6. Both new files also get a PDF export alongside them.

**Step 6 - report back.** The computed hours per row, which rows were corrected and why, the names of the files created, and how the tracking file changed.

## Notable implementation details

A few things the workflow does deliberately, documented in `references/timesheet-processing.md`:

- **Exclusive break-rule boundaries.** A rule with tiers like "under 6h no break, under 9h 30 min, otherwise 45 min" has to treat each boundary as an exclusive upper bound, or a day of exactly 9.00 hours lands one tier too low. It's an easy off-by-one to introduce, it only ever shows up on boundary values, and it costs real money in both directions. This is called out wherever the rule is applied, including a check that your own template's formula uses the same convention.
- **Exact digit position via pixel analysis, not guesswork.** Photos of paper forms are usually slightly perspective-distorted, so a fixed row height drifts down the page. The workflow locates the actual ink position of the handwritten digit by counting dark pixels in a narrow window, which also distinguishes a printed grid line (dark across the full width) from handwriting (dark in a few places only).
- **Embedded fonts for PDF annotations.** Red correction text inserted with a non-embedded base-14 PDF font can look perfectly fine in the tool that wrote it, then come out garbled in a different PDF tool downstream, for example when invoicing software merges the file later. The workflow always embeds a real font file and verifies the text round-trips back out as inserted.
- **Two independent numbering sequences.** A document-facing number and a purely internal count are easy to conflate, and conflating them means a wrong number on a real billing document. They're kept as separate concerns with separate labels throughout.

## Limitations

- **Handwriting is read by image analysis, so check it before billing.** The correction copy and the clean timesheet are a first pass, not an authority. Read both before they go anywhere near an invoice or a payroll run.
- **No Excel COM automation in a sandboxed shell.** Converting the generated `.xlsx` to PDF through Excel automation can fail in a sandboxed or restricted-token shell on Windows even with Excel correctly installed. That's a boundary of the environment, not a setting to fix. The skill tests for it and asks you to export that one file yourself instead of silently skipping it or retrying forever.
- **Developed on Windows.** The numbering and image workflows aren't platform-specific, but the PDF-export note above is.
- **The bundled templates are examples.** They carry placeholder names and a sample break rule so you can see the expected shape. Point the config at your own spreadsheet for real work.

## Background

This came out of a real trade business, not a blank page. Every timesheet that arrived raised the same two questions in the same order - which number does this one get, and do the hours on it actually add up - and the second question was the one that got skipped when things were busy, which is precisely when a wrong break deduction is most likely to be written down and then billed.

Automating the arithmetic was the easy half. The half worth documenting was everything the first few runs got subtly wrong: a break rule that behaved correctly except on exact boundary values, annotations that were readable in one PDF viewer and garbage in the next, and correction marks that drifted off their row because a photographed form isn't a flat scan. Those three are written up in the workflow itself rather than silently fixed, because anyone adapting this to their own forms will meet all three.

Generalising it afterwards meant removing every trace of the original business: company names, folder structure and the fixed spreadsheet layout all became config, and the setup interview replaced the assumptions.

## No warranty

The MIT license below covers this in legal terms; in plain ones: this skill reads handwriting from photographs, does arithmetic on it, assigns numbers that end up on real billing documents, and writes files into your project folders. It is a competent first pass and nothing more. Check the corrections, check the totals, check the number it assigned. What you invoice is yours, not the skill's.

## What's in this repo

- `SKILL.md` - the entry point, loaded by Claude Code when this skill is relevant
- `references/setup.md` - the first-use interview and the config schema
- `references/numbering.md` - the full numbering workflow
- `references/timesheet-processing.md` - the full timesheet review and correction workflow
- `config.example.json` - a filled-in sample configuration
- `assets/tracking-template.xlsx` - bundled example project-tracking spreadsheet
- `assets/timesheet-template.xlsx` - bundled example timesheet spreadsheet

## Author

Dennis Frösch - [@DeFroe](https://github.com/DeFroe)

## License

MIT - see [LICENSE](LICENSE).
