# First-use setup

Triggered from SKILL.md Step 0 when no config file exists yet at
`%USERPROFILE%\.claude\skill-config\craft-timesheet-numbering-correction\config.json`.

## Interview

Ask in two batches: a bundled multiple-choice batch first (via the question tool, all in one
call), then free-text follow-ups depending on the answers.

**Batch 1 - bundled choice questions:**

1. Do you need a **second, internal-only** numbering sequence in addition to the main document
   number? (e.g. to separately count how many originals came from a specific subcontractor,
   without that count ever appearing on a document). Yes / No.
2. Which timesheet template do you want to use: the **bundled example** template
   (`assets/timesheet-template.xlsx`) or **your own** existing spreadsheet?
3. Which break-time rule do you want: the **bundled example preset** (a simple three-tier rule -
   shown to the user for review before accepting) or **your own thresholds**?

**Batch 2 - free text, based on Batch 1 answers:**

- Base folder: the root path under which all project folders live.
- Naming pattern for the per-project tracking file (e.g. `Tracking_{project}.xlsx`). Ask whether
  any characters in project names need replacing for the filename (e.g. a comma becoming an
  underscore) and record the rule if so.
- Labels for the two parties that appear on a timesheet (e.g. "Client" / "Subcontractor", or
  whatever terms the user's business actually uses - this becomes `party_a_label` /
  `party_b_label`). Also ask what should appear in the party-B field for work done by the user's
  own staff rather than a subcontractor (default suggestion: `"internal"`).
- If "own template" was chosen: ask for its path, then open it read-only with openpyxl, scan
  column A/B (or the first non-empty column) for label-like text near the top, and propose a
  `template_mapping` (see schema below) for confirmation/correction - don't guess coordinates
  blindly. If the file is currently open elsewhere, reading it may still work (openpyxl only
  needs write access when saving), but writing to it later will need it closed first.
- If "own thresholds" was chosen: ask for the hour boundaries and corresponding break lengths,
  as a list from lowest to highest. Confirm the boundary is treated as an **exclusive** upper
  bound on each tier (i.e. "up to but not including this many hours") - this is deliberate: it
  avoids an off-by-one at exact boundary values, which is a real mistake this skill's design
  history is based on.
- Filename pattern for generated documents, with placeholders `{party}`, `{project}`,
  `{doc_number}` (default suggestion:
  `{party}_Timesheet_{project}_ORIG_zu_{doc_number}`).

## Confirm and persist

Show a short summary of everything gathered, ask for confirmation or corrections, then write it
to:

```
%USERPROFILE%\.claude\skill-config\craft-timesheet-numbering-correction\config.json
```

Create the parent folder if it doesn't exist. This location is deliberately **outside** the
skill's own folder, so a future update/re-clone of the skill (e.g. via `git pull`) can never
overwrite or delete it.

## Config schema

See `config.example.json` in the skill root for a filled-out example. Fields:

| Field | Type | Meaning |
|---|---|---|
| `base_folder` | string | Root path containing one folder per project. |
| `party_a_label` / `party_b_label` | string | Labels for the two parties on a timesheet. |
| `internal_party_value` | string | What to write in the party-B field for the user's own staff. |
| `secondary_sequence_enabled` | boolean | Whether a second, internal-only numbering sequence exists. |
| `tracking_filename_pattern` | string | Filename pattern per project, with `{project}` placeholder. |
| `project_name_replacements` | object | Optional map of characters to replace when building filenames from a project name, e.g. `{", ": "_"}`. |
| `break_rules` | array of `{max_hours, break_hours}` | Ordered low→high. `max_hours: null` on the last tier means "and above". `max_hours` is an **exclusive** upper bound. |
| `template_source` | `"bundled"` \| `"custom"` | Which timesheet template to use. |
| `template_path` | string (if `custom`) | Path to the user's own template. |
| `template_mapping` | object | Cell coordinates for the fields the workflow needs to fill (see `references/timesheet-processing.md` for the field list). |
| `doc_filename_pattern` | string | Pattern with `{party}`, `{project}`, `{doc_number}` placeholders. |

## Re-running setup

If the user says something like "reset setup" or "reconfigure", re-run the full interview and
overwrite the existing config file after confirmation - don't merge silently with old values
unless the user asks to keep some of them.
