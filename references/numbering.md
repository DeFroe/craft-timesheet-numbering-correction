# Determining the next free running number

Assumes Step 0 (config loaded) already happened — see SKILL.md. All values below (folder names,
labels, filename patterns) come from the config, not from hardcoded examples.

## 1. Find the project folder

When the user names a project, look under `config.base_folder` for a matching folder (substring/
fuzzy match is fine, e.g. "Diez" → "Turnhalle am Wirt, Diez"). If there are multiple matches or
no clear match: list the candidates found and ask, rather than guessing.

## 2. Read the tracking file

`.xlsx` is a binary file; the Read tool can't open it. Find the file in the project folder using
`config.tracking_filename_pattern` (with `{project}` substituted, and `config
.project_name_replacements` applied if set). If the exact computed name isn't found, search the
folder for any file whose name matches the pattern loosely, rather than failing outright.

Read it with Python + openpyxl (`python`, not `python3`, in most environments):

```python
import openpyxl
wb = openpyxl.load_workbook(r"<path>", data_only=True)
```

If opening for writing later fails with `PermissionError`, the file is likely open elsewhere
(e.g. in Excel) — ask the user to close it rather than working around the lock.

## 3. Master data

Row numbers can vary slightly between files — look for label text in column A (or the first
non-empty column) rather than trusting fixed coordinates blindly. At minimum, a tracking file
typically has:

- Party A (e.g. client) name + address
- Project name
- Party B (e.g. subcontractor) name + address, if `secondary_sequence_enabled`
- Any rate/price fields relevant to the business

## 4. Running-number tables

Locate the table(s) starting at a header row like "No." / "Document No." (and, if
`secondary_sequence_enabled`, a second table for the internal sequence). Numbers may be
zero-padded (e.g. "008") or not (e.g. "5") — keep whatever format the most recent entry uses when
proposing the next one.

## 5. Determine the next free number

For each configured sequence independently: find the last row whose number cell is filled, and
propose last + 1 in the same format. If no number has been assigned yet, start at 1 (or the
zero-padded equivalent, e.g. "001", if that's the established format). If the row index and the
actually-entered number diverge (gaps, duplicates, implausible jumps) or the pattern isn't clear:
name the inconsistency and ask the user rather than deciding unilaterally.

**The two sequences are independent and serve different purposes** (only relevant if
`secondary_sequence_enabled`):

- **Primary number**: always used in the filename and on the document itself, regardless of
  whether the underlying work came from a subcontractor or the user's own staff. Every document
  created and placed in the project folder gets the next free primary number.
- **Secondary number**: purely internal bookkeeping — how many originals from a given
  subcontractor have been processed. Never appears in a filename or on a document; only tracked
  inside the tracking file.

Getting these two mixed up is an easy, consequential mistake (a wrong number ending up on a real
document that then has to be renamed by hand) — double-check which one is being assigned whenever
a number is used for a document.

## 6. Report back

Tell the user: project name, both parties, the determined next primary number (for the document
about to be created), and — if relevant — the next secondary number (for internal tracking).
Name both and briefly explain which is which when a timesheet is being processed.

## Scope

For a pure number lookup: don't write to the tracking file and don't generate a new document from
a template — that's a separate, explicitly-requested step (see `timesheet-processing.md`). When
in doubt about format or mapping: ask rather than guess — these numbers end up on real business
documents.
