# Reviewing an original timesheet and generating a clean copy

Builds on the base workflow (find project folder, read tracking file, determine the next free
number - see `numbering.md`). Triggered when the user wants a photographed/scanned handwritten
timesheet reviewed and turned into a clean digital timesheet.

Unlike the pure number lookup, this is **writing**: it creates new files in the project folder
and adds an entry to the tracking file. Before creating anything, summarize what will be created
(filenames, which number will be assigned) so the user can follow along - especially if a file
with the same name already exists.

## 1. Find the original

Look in the project folder for a not-yet-processed PDF or image file (e.g. a photo of a
handwritten timesheet). If there are multiple candidates or it's unclear which file is meant,
ask rather than guessing.

**Internal-staff special case:** if `secondary_sequence_enabled` and the original is marked as
work by the user's own staff rather than a subcontractor, handle it computationally the same way
(same break rule, same comparison against entered values). If a calculation error is found (see
step 3), a red-annotated correction copy is created for this case too, exactly as for a
subcontractor original.

**Renaming the original:** if the original doesn't yet have a filename matching the configured
`doc_filename_pattern`, rename it directly:

```
config.doc_filename_pattern with {party}=<party name>, {project}=<project>, {doc_number}=<primary number>
```

The file is renamed, not duplicated; keep its original extension. `{doc_number}` is the same
primary number that will also be used on the generated timesheet (step 5).

**Filename collision when a correction copy is also needed:** if a calculation error additionally
produces a correction copy (step 4), it would get the same name as the just-renamed original.
To avoid that, append `_corrected` to the correction copy's filename before its extension in this
case. This suffix only matters when the original gets renamed directly (the internal-staff case,
or any case where step 1's renaming applies) - when the original keeps its raw scanner filename
and only the correction copy uses the pattern name, there's no collision and no suffix is needed.

## 2. Compute worked hours and break

For each row of the original, read the start/end time (handwritten is fine, via image analysis).
Worked hours = end time - start time, in decimal hours (e.g. 8:00 to 16:30 → 8.50 h).

Apply `config.break_rules` (ordered low→high, each `max_hours` an **exclusive** upper bound):
worked hours below the first tier's `max_hours` get that tier's `break_hours`, and so on, with
the last tier (`max_hours: null`) applying at and above the highest listed boundary.

**Watch the exact boundary values carefully** - because `max_hours` is exclusive, a value exactly
equal to a boundary belongs to the *next* tier up, not the one below. This is a deliberate design
choice specifically to avoid an easy off-by-one mistake at boundary values.

Net hours = worked hours - break, per this rule. This rule is the reference the original is
checked against, regardless of what's written on it.

## 3. Compare the original against the computed values

If the break or resulting net hours written on the original differ from step 2's calculation,
the row needs correction.

## 4. Create a correction copy

Create a copy of the original (same file format: PDF stays PDF, image stays image) and, for each
incorrect row:

- Insert the correct value in **red**, in square brackets, **to the left** of the wrong original
  value. If there's no room to the left (the digit sits right against a column border),
  place it to the right instead, or in the blank row directly below if the layout has one - note
  this deviation to the user briefly.
- Strike through the wrong original value with a **red line**.

Example row (original wrongly shows 1.00 h break instead of the correct 0.50 h):

```
8:00 - 16:30  =  8.50      [-0.50]  ~~-1.00~~      [8.00]  7.50
```

(`[-0.50]` and `[8.00]` in red, `-1.00` struck through in red; `~~..~~` here is just a
placeholder for "struck through", not literal Markdown in the target document.)

**Implementation:**
- Image files: Python with Pillow (`PIL.ImageDraw`) - draw red text and a red line.
- PDF files: Python with PyMuPDF (`fitz`) - draw red annotations/freehand lines.

**Always use an embedded font for `insert_text`, never a base-14 font like `fontname="helv"`**:

```python
page.insert_text(
    point, text, fontsize=11, color=(1, 0, 0),
    fontfile=r"C:\Windows\Fonts\arial.ttf", fontname="F0-embedded",
)
```

A non-embedded base-14 font can get mis-decoded into garbled characters by other PDF tools
downstream (e.g. when a separate invoicing/merging tool processes the file later) - even though
it displays fine in the tool used to create and check it. An embedded font (Identity-H encoding,
correct ToUnicode mapping) stays correctly readable/extractable regardless of which tool
processes the PDF next. Verify by extracting the inserted text back out (`page.get_text("dict")`)
and confirming it round-trips to the exact string inserted, not just by looking at a render from
the same library that wrote it.

**Locating the exact digit position (don't just estimate):** photos of paper forms are often
slightly perspective-distorted, so guessing the row height isn't reliable enough:

1. For the target column, find the actual **ink position** of the digit via pixel analysis: in a
   narrow x-window around the expected digit, for a y-window around the rough row estimate, count
   pixels below a brightness threshold. A full grid line is dark across the entire column width
   (high count); a single digit is dark at only a few x-positions (low count) - this distinguishes
   grid lines from handwriting. The mean of the matching y-values is the row's vertical center.
2. Use that vertical center for the annotation (a right/left-aligned text anchor, not a fixed
   offset that drifts with varying row spacing).
3. Don't choose too small a font size (roughly 12px minimum at a photo resolution around
   960×1280). After drawing, check the result with a **heavily zoomed-in crop** (4-6x) of the
   affected row before reporting it as done - at normal preview size, thin red annotations next
   to dark handwriting can visually disappear even though they were drawn correctly.

**Correction copy filename:**

```
config.doc_filename_pattern with {party}=<party name>, {project}=<project>, {doc_number}=<primary number>
```

with the same file extension as the original.

- `{party}` and `{project}` come from the tracking file's master data (see `numbering.md`); keep
  spacing/special characters as they appear in the original.
- `{doc_number}` is always the next **primary** number (see `numbering.md`) - regardless of
  whether the source data came from party B or the user's own staff; the secondary number is
  never used for a document filename, only tracked internally.
- Creating this copy assigns the primary number: add the corresponding entry (index, number,
  date) to the tracking file. If the data came from party B, also determine and record the next
  secondary number (purely internal) - it doesn't appear in the filename or on the document.

## 5. Generate the clean timesheet

Copy `config.template_path` (or the bundled `assets/timesheet-template.xlsx` if
`template_source` is `"bundled"`) and fill it in using `config.template_mapping`. A timesheet
template typically needs:

- Party A address block.
- Party B name (name only - no address, even if one is on file - unless it's the internal-staff
  case, where this field instead just gets `config.internal_party_value`).
- Project name, document number (the primary number), date.
- One row per work entry: date, description/worker name, start time, end time, a person-count
  multiplier (not a name field - if a row is one person, this is 1), break (formula), net hours
  (formula), travel time.
- **A break-formula boundary check**: before filling in rows, verify the template's break formula
  uses the same **exclusive** upper-bound convention as `config.break_rules` (`<`, not `<=`, at
  each boundary). Bundled/older templates sometimes get this backwards, which silently
  miscalculates only the rows that land exactly on a boundary value - worth spot-checking even if
  the template "looks" already correct.
- **Travel time as an expression, not literal**: originals sometimes contain an arithmetic
  expression instead of a plain number (e.g. "4-1" meaning 4 minus 1, or "2+2"). Evaluate it and
  enter the **result** as a number, not the expression text. Only leave a value unevaluated (and
  flag it to the user) if it's genuinely ambiguous or not a simple calculation.
- **Fill every summary cell, not just the hours column**: both the hours sum and the travel-time
  sum need a formula, and if the template has a separate "total" cell that's supposed to combine
  hours and travel time, make sure it actually references both - not just the hours sum. Templates
  can silently have this wrong; verify by computing an expected total by hand and comparing.
- **A duplicated date field**: many timesheet layouts repeat the date near a signature line at
  the bottom, separate from the header date field - easy to miss because it can sit in a row that
  looks empty at a glance. Fill both.
- **Don't recreate embedded images (signatures/logos) programmatically**: if the template already
  has a stamp/signature image baked in, copying the template preserves it automatically -
  inserting one again via code risks misaligning or duplicating it.

**Keep the one-row-per-entry structure, even for many rows.** Don't switch to a condensed
summary format (e.g. combining several days/people into one descriptive line) even if the
template runs out of rows on one sheet - add a second sheet/page with the same document number
instead.

**Date in the document-number field and in the tracking file:** if the tracking file already has
a date recorded for this number, use that - not today's date. If none is recorded and the
original itself has no signature/end date either, fall back to today's date and say so
transparently.

Save the file in the project folder (filename analogous to the correction copy, with e.g.
"Timesheet" instead of "ORIG" in the name, unless the user specifies otherwise).

## 6. Export to PDF

Both the correction copy (if created) and the new timesheet get an additional PDF export with
the **same filename**, just with a `.pdf` extension.

**Image correction copy → PDF**: direct and reliable with Pillow, no external programs needed:

```python
from PIL import Image
Image.open(png_path).convert("RGB").save(pdf_path, "PDF")
```

**Timesheet (XLSX) → PDF**: requires Excel COM automation (`pywin32`). **Known limitation**: in a
sandboxed/agent shell environment on Windows, COM activation of Excel can fail entirely
(`REGDB_E_CLASSNOTREG` / `0x80070490`) even though Excel is correctly installed and a plain
process launch works fine - this happens when the shell process runs under a restricted access
token that DCOM activation checks reject. This is not a fixable configuration; it's a hard
boundary of that kind of sandboxed environment. Test with a small COM call before attempting the
real export; if it fails, tell the user directly that xlsx→PDF export isn't possible from this
environment and ask them to export it once themselves (File → Export → Create PDF/XPS, same
filename) - don't silently skip it or keep retrying.

## 7. Report back

Summarize for the user: computed net hours per row, which rows were corrected and why, the names
of newly created files (with the assigned number, including PDF exports or a note if the xlsx→PDF
export wasn't possible in this environment), and how the tracking file was updated (primary-number
entry, and the internal secondary-number entry if the source was party B).
