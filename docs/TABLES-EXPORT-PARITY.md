# Tables export parity: `--export-pdf` hook spec (amendment)

Amends [EXPORT-PARITY-SPEC.md](EXPORT-PARITY-SPEC.md), whose owner direction
puts xlsx out of scope ("spreadsheets are not an important published
rendered artifact"). This amendment carves out the smallest possible
exception: a headless `--export-pdf` hook on the Tables canvas/grid render
path, scoped to opting in the three both-green screenshot fixtures
(`tables/values`, `tables/merged`, `tables/frozen` — Tier A and Tier B
green in `tools/render-lab/baseline.json`), so an export diff measures the
exporter and not the on-screen renderer, per the existing spec's opt-in
rule. It does **not** re-scope xlsx generally, and it does not touch the
screenshot-parity thresholds.

Spec only. No implementation in this change.

## 1. What PR #1108 established (the pattern to follow)

Commit `357d156` ("feat(letters,decks): headless --export-pdf hooks on the
canvas paths") implemented spec item 1 of EXPORT-PARITY-SPEC.md for Letters
and Decks. The Tables hook mirrors it exactly:

1. `tables/src/main.rs`: register a real `--export-pdf OUT` GApplication
   option (`suite_common::render_dump::EXPORT_PDF_FLAG`,
   `OptionArg::Filename`), parse it display-free in
   `connect_handle_local_options` via
   `suite_common::render_dump::export_pdf_path` (setting
   `GTK_OFFICE_TEST_MODE=1` and `GTK_OFFICE_EXPORT_PDF`), and in
   `connect_open` call `suite_common::render_dump::schedule(app)` plus,
   when the env var is set, `schedule_export(app)`. Also handle the
   no-input-file case (`--export-pdf` with no file: message on stderr and
   quit), as both apps do.
2. `tables/src/window.rs`: add a `test-export-pdf` action that writes the
   PDF named by `GTK_OFFICE_EXPORT_PDF` through the canvas path, then
   `schedule_export` activates it after the window settles and quits.
3. `suite-common/src/render_dump.rs`: **no changes** — `EXPORT_PDF_ENV`,
   `EXPORT_PDF_FLAG`, `export_pdf_path`, and `schedule_export` are reused
   as-is (arg parsing is already unit-tested there).

Faithfulness precedents to keep: Letters reuses its `printing.rs` Cairo
path (`Typeset::draw_page`, the same pages the view shows); Decks draws the
canvas through `draw_slide_in` with `Chrome::Show`, explicitly **not** the
Typst path in `decks/src/export.rs`, at the model's own size
(`slide_page_size_pt`, 720x405pt), restricted to PDF 1.4 so the page tree
stays countable. The Tables hook is the canvas path, not any Typst or
print-pipeline path, for the same reason.

## 2. What the hook draws: the render-dump rect, not the viewport

**Decision: full used-range rect (clipped to the viewport), single PDF
page — not the current viewport.** Rationale:

- The existing `test-render-dump` action in `tables/src/window.rs`
  restores the saved view (`grid_render::restore_saved_view`), computes
  the used range's far edge
  (`sheet.used_extent()` → `col_x + col_width`, `row_y + row_height`),
  and snapshots the rect `(0, 0, w.min(aw), h.min(ah))` — the used range
  clipped to the drawing area. The export draws **that same rect**: it is
  "the rendered sheet", and for the three in-scope fixtures the used range
  fits the window, so rect and viewport coincide in practice.
- The rect must be computed **after** `restore_saved_view`, as the
  render-dump does. The `tables/frozen` fixture is saved scrolled
  (`pane.topLeftCell = E20`, `freeze_panes = B2` in
  `tools/render-lab/fixtures.py`): without the restore, the grid shows
  A1:I39 instead of row 1 + column A beside E20:I39 and most words go
  missing. Calc-side print titles (`print_title_rows/cols`) and print area
  (`E20:I39`) are the print form of that freeze and need no app-side
  equivalent — restoring the saved view already shows what Calc prints.
- Drawn with `show_gridlines = false`. Both sides agree on this today:
  `fixtures.py` sets `sheet.print_options.gridLines = False` (gridlines
  are view furniture; Calc prints them black, screens draw them faint, and
  a printed gridline hides a thin cell border), `print_options.headings =
  True`, and `capture.py` seeds Tables with `show-gridlines=false`. (The
  `test-render-dump` comment in `window.rs` saying "headings and gridlines
  on" is stale; the seeded setting and the fixtures govern.)
- Editing chrome stays out exactly as on screen: `draw_grid` already hides
  the selection wash/outline when `render_dump::active()` (the export sets
  `GTK_OFFICE_TEST_MODE`), the same way the caret and selection never
  reach the reference.

New function (name indicative, not mandated): `render_sheet_pdf` next to
`draw_grid` in `tables/src/grid_render.rs` — create a Cairo `PdfSurface`
at the rect size in points (pixels × 72/96), `restrict` to PDF 1.4 like
Decks, replay `draw_grid` onto it with the restored scroll offsets and
`show_gridlines = false`, one `show_page`. Empty sheet (no used extent)
is an error, mirroring Decks' "no slides to export" rather than an empty
PDF. Unit tests mirror Decks': real `%PDF` header, one page
(`/Type /Page` count minus `/Type /Pages`), `/MediaBox` equal to the rect
in points, empty-sheet error; text landing is judged by rasterising
(`pdftoppm`), not by grepping the PDF, since Cairo may subset fonts.

## 3. Paper-fidelity rule, Tables form

The existing spec's rule ("the PDF page size must be the document's size")
becomes: **one PDF page sized to the export rect in points.** There is no
model slide size to reuse (Decks has one; Tables has none), and there is
no pagination — a multi-page sheet is out of scope (section 5). The
`page_count_match` budget in `export_compare.py` applies unchanged: the
three in-scope fixtures are small (5x4 values, one merged block, E20:I39
with titles) and Calc is expected to print each on a single page; if an
implementation-time capture shows Calc paginating one of them, that is a
fixture-scoping finding to report, not a licence to paginate in the hook.

## 4. Lab wiring and metric/threshold conventions (unchanged)

- Reference side needs nothing new: `lo_render.py` converts any fixture
  file (`soffice --convert-to pdf`) and rasterizes with `pdftoppm -r 96`
  to `lo-<n>.png`; the Calc PDFs of the equivalent `.xlsx` files flow
  through the same invocation.
- `export_render.py`: add `"tables"` to `EXPORT_APPS` (the "manifest bug"
  guard flips: an `export: true` tables fixture becomes legal) and run the
  binary under Xvfb with the same Tier A window seeding, so the settled
  layout the exporter draws is the one the screenshots judge.
- `fixtures.py`: add exactly `tables/values`, `tables/merged`,
  `tables/frozen` to `EXPORT`; `baseline-export.json` seeds each as
  `missing` until measured.
- `export_compare.py`: untouched — same `compare.align` resampling, same
  `compare.verdict` budgets (all must hold for green: SSIM ≥ 0.75, words
  ≥ 0.90 with ref_words ≥ 3, `lost_lines == 0`, displacement ≤ 6.0pt,
  colours ≥ 0.90, and for tables |scale − 1| ≤ 0.10, plus
  `page_count_match`), same tier name `export`, same separate-baseline
  ratchet. The sparse-tables OCR path (`ocr_words(img, sparse=true)`,
  already used for `app == "tables"`) applies to both sides automatically.
- `tests/test_export_parity.py` (`ExportOptInTest`): the
  `assertIn(app, ("letters", "decks"))` gate gains `"tables"` with a
  comment that only the three both-green fixtures may opt in, and the
  size cap (`≤ 35`) is re-based to the new opt-in count. The honesty rules
  apply unchanged: look at the images, fix the exporter or the metric —
  never the thresholds.

## 5. Out of scope (explicitly not this hook)

- Pagination / multi-page sheets, print scaling, page setup and margins.
- Any other tables fixture (`wrap-text` and `number-formats` are amber on
  screen; charts, conditional formatting and the rest stay screenshot-only).
- The Typst path, threshold or metric changes, `baseline.json`, and the
  `tables/wrap-text` (#920) Excel-vs-Calc decision, all per the existing
  spec.
- General xlsx re-scoping: EXPORT-PARITY-SPEC.md's "xlsx entirely" line
  stands apart from this amendment's three-fixture carve-out.

## Acceptance

Per-fixture `ours-<n>.png` vs `lo-<n>.png` export verdicts in CI for
`tables/values`, `tables/merged`, and `tables/frozen`, tracked in
`baseline-export.json` under the unchanged ratchet (regression fails;
improvement without `--update-baseline` fails stale), with the hook code
following the #1108 pattern in section 1 and the rect semantics in
section 2.
