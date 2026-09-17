---
name: craft-timesheet-numbering-correction
description: Tracks the next free running document number for a project (two independent sequences - a primary number used on documents, and an optional secondary number for internal-only counting), and processes photographed/scanned handwritten timesheets - computes worked hours, applies a configurable break-time rule, compares against the entered values, produces a red-annotated correction copy for mismatches, and generates a clean digital timesheet from a template. Use when the user asks for the next document number, wants to prepare a new timesheet for a project, wants an original scanned timesheet reviewed/corrected, or mentions a project tracking file / running number list. Requires a one-time setup interview on first use (see references/setup.md) to learn the user's folder layout, terminology and rules.
---

# Craft Timesheet Numbering & Correction

Helps a craft/trade or service business (construction, landscaping, cleaning, any subcontractor-
based trade) with two related jobs for a project-based file layout:

1. **Numbering** - read a per-project tracking file and determine the next free running document
   number, in one or two independent numbering sequences.
2. **Timesheet correction** - read a photographed/scanned handwritten timesheet, compute the
   correct worked hours under a break-time rule, compare against what was actually entered, flag
   and visually correct mismatches, and produce a clean digital timesheet.

This skill is **generic and configuration-driven**. Nothing below hardcodes a company name, a
folder path, or a fixed spreadsheet layout - those live in a config file created during a
one-time setup interview.

## Step 0: Load or create configuration (always first)

Before doing anything else, check for the config file:

```
%USERPROFILE%\.claude\skill-config\craft-timesheet-numbering-correction\config.json
```

- **Exists and has all required fields** → read it and use those values for the rest of this
  skill. Do not re-run the interview.
- **Exists but is missing required fields** → ask only for the missing fields (partial repair),
  don't repeat the whole interview.
- **Does not exist** → briefly tell the user this looks like the first use, then run the setup
  interview described in [references/setup.md](references/setup.md). Show a summary of the
  gathered values for confirmation, then write the config file (creating the folder if needed).
  Only after that, continue with the user's original request.
- The user can say **"reset setup"** (or similar) at any time to re-run the interview and
  overwrite the existing config.

Required config fields are documented in `references/setup.md` and mirrored in
`config.example.json` in this skill folder.

## Folder layout the config describes

```
<base_folder>\
  <template files>                      <- blank templates (bundled or user-supplied)
  <project-folder>\                     <- one folder per project
    <tracking_filename_pattern>         <- running-number tracking file for this project
    ...                                 <- e.g. scanned timesheet photos dropped here
```

## Workflow: numbering

See [references/numbering.md](references/numbering.md) for the full, detailed workflow:
finding the project folder, reading the tracking file, locating the master-data block, reading
the running-number tables, determining the next free number for each configured sequence, and
what to report back.

This part is **read-only**: it never writes to the tracking file and never creates a new
timesheet document by itself. Writing is a separate, explicitly requested step (see below).

## Workflow: timesheet correction

See [references/timesheet-processing.md](references/timesheet-processing.md) for the full,
detailed workflow: finding the original, computing worked hours and breaks, comparing against
the entered values, producing a red-annotated correction copy (including the pixel-calibration
technique and the embedded-font requirement for PDF annotations), generating a clean digital
timesheet from the template, exporting to PDF, and updating the tracking file.

This part is **writing**: it creates new files in the project folder and appends an entry to the
tracking file. Before creating anything, summarize what will be created (filenames, which number
will be assigned) so the user can follow along - especially if a file with the same name already
exists.

## Important

- For a pure number lookup: don't write to the tracking file and don't generate a new timesheet
  from the template, as long as only the next number was asked for.
- When in doubt about format, mapping, or an ambiguous/gapped number sequence: ask rather than
  guess - these numbers end up on real billing documents.
