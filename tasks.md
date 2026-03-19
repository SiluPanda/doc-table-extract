# doc-table-extract — Task Breakdown

This file tracks all implementation tasks derived from SPEC.md. Each task is a granular, actionable unit of work.

---

## Phase 1: Project Scaffolding and Type Definitions

- [ ] **Install runtime dependencies** — Add `pdfjs-dist` as a runtime dependency in `package.json`. | Status: not_done
- [ ] **Install dev dependencies** — Add `typescript`, `vitest`, and `eslint` as dev dependencies in `package.json`. Verify `vitest run` works with an empty test suite. | Status: not_done
- [ ] **Add peer dependencies** — Add `tesseract.js` and `canvas` as optional peer dependencies in `package.json` with `peerDependenciesMeta` marking them as optional. | Status: not_done
- [ ] **Configure CLI binary** — Add `"bin": { "doc-table-extract": "./dist/cli.js" }` to `package.json`. | Status: not_done
- [ ] **Create directory structure** — Create all directories specified in SPEC Section 21: `src/detect/`, `src/extract/`, `src/pdf/`, `src/image/`, `src/output/`, `test/fixtures/`, `test/unit/`, `test/integration/`. | Status: not_done
- [ ] **Define core types (types.ts)** — Implement all TypeScript interfaces from SPEC Section 14: `TableInput`, `ExtractOptions`, `LatticeOptions`, `StreamOptions`, `OCROptions`, `LLMOptions`, `DetectOptions`, `ExtractedTable`, `Cell`, `TableMetadata`, `TableRegion`, `TableExtractor`. | Status: not_done
- [ ] **Define internal types** — Define internal types not in the public API: `TextElement` (text, x, y, width, height, fontName, fontSize), `LineSegment` (x1, y1, x2, y2, lineWidth), `GridCell`, `Intersection`, and `TableCandidate`. | Status: not_done
- [ ] **Define default configuration constants** — Create a defaults/constants module with all default values from SPEC Section 15: mode='auto', minConfidence=0.5, headerRows='auto', mergeMultiPageTables=true, preserveCellNewlines=false, OCR defaults, lattice defaults, stream defaults. | Status: not_done
- [ ] **Set up public API exports (index.ts)** — Export `extractTables`, `extractTablesFromPDF`, `extractTablesFromImage`, `detectTableRegions`, `createExtractor`, and all public types from `src/index.ts`. | Status: not_done

---

## Phase 2: PDF Parsing Layer (`src/pdf/`)

- [ ] **Implement PDF parser wrapper (pdf-parser.ts)** — Create a wrapper around `pdfjs-dist` that loads a PDF from a file path, Buffer, or URL using `getDocument()`. Provide methods to get page count, get a specific page, and clean up resources. Handle encrypted PDFs via the `password` option. | Status: not_done
- [ ] **Implement text element extraction (text-elements.ts)** — Extract all text content items from a PDF page via `page.getTextContent()`. Normalize each `TextItem` into the internal `TextElement` type, computing x/y position from the transformation matrix `[a, b, c, d, e, f]` (e=x, f=y) and fontSize from `Math.sqrt(a*a + b*b)`. Normalize to top-to-bottom reading order. | Status: not_done
- [ ] **Implement font analyzer (font-analyzer.ts)** — Analyze font metadata from `pdfjs-dist`'s `commonObjs`. Detect bold fonts by checking font name for "Bold", "Bd", "Heavy", "Semibold", or "Medium". Extract font size. Provide helpers for comparing font weight and size between rows. | Status: not_done
- [ ] **Implement line extractor (line-extractor.ts)** — Extract line segments from a PDF page's operator list via `page.getOperatorList()`. Parse `OPS.constructPath` operators to get moveto/lineto/rectangle commands. Apply the current transformation matrix to get page coordinates. Decompose rectangles into four line segments. Treat thin rectangles (width or height below 3 points) as thick lines. Output `LineSegment[]`. | Status: not_done
- [ ] **Unit test: text element extraction** — Test that `TextElement` objects are correctly computed from mock `TextItem` data with known transformation matrices. Verify x, y, width, height, fontName, fontSize fields. | Status: not_done
- [ ] **Unit test: font analyzer** — Test bold detection for font names containing "Bold", "Bd", "Heavy", "Semibold", "Medium" and non-bold font names. Test font size comparison. | Status: not_done
- [ ] **Unit test: line extractor** — Test extraction of horizontal lines, vertical lines, rectangles, and thin rectangles from mock operator lists. Verify coordinate transformation is applied correctly. | Status: not_done

---

## Phase 3: Table Detection (`src/detect/`)

- [ ] **Implement ruled line detection (line-detector.ts)** — From extracted line segments, classify lines as horizontal (|y1-y2| < tolerance, length > minLineLength, lineWidth < maxLineWidth) or vertical (|x1-x2| < tolerance, height > minLineLength). Snap lines with y-coords (horizontal) or x-coords (vertical) within snapTolerance to the same coordinate. Discard diagonal lines and curves. | Status: not_done
- [ ] **Implement grid detection within line-detector** — After line classification, detect grid patterns: find intersections of horizontal and vertical lines, verify at least 2 horizontal and 2 vertical lines intersect forming a rectangular lattice. Compute the bounding box of each detected grid. Support multiple non-overlapping grids on a single page. | Status: not_done
- [ ] **Implement text cluster detector (text-cluster-detector.ts)** — Group text elements by y-coordinate into row groups (within one line-height tolerance). For each row group, collect x-coordinates of text element left edges. Identify recurring x-coordinate values across multiple rows (column alignment). Require at least 3 consecutive aligned rows with at least 2 columns. Compute bounding box from participating text elements. Distinguish from prose by verifying horizontal gap between columns is at least 2x average word spacing. | Status: not_done
- [ ] **Implement confidence scoring (confidence.ts)** — Compute table region confidence from weighted signals per SPEC Section 5.4: grid lines present (0.30 weight), column count consistency (0.20), row count (0.15, scales 0.3 for 2 rows to 1.0 for 5+ rows), column spacing consistency (0.15), cell fill ratio (0.10), font consistency (0.10). Normalize to 0.0-1.0 range. | Status: not_done
- [ ] **Implement detection coordinator (detector.ts)** — Orchestrate detection across pages. For each page: run line detection (for lattice candidates), run text cluster detection (for stream candidates). Apply filtering rules from SPEC Section 5.3: minimum size (50x50 points), minimum cells (2 rows, 2 columns), overlap resolution (>50% overlap keeps higher confidence), confidence threshold (minConfidence). Order detected regions top-to-bottom, left-to-right. Return `TableRegion[]`. | Status: not_done
- [ ] **Unit test: line detection** — Provide mock line segments. Verify horizontal/vertical classification, snapping, and diagonal line rejection. | Status: not_done
- [ ] **Unit test: grid detection** — Provide sets of horizontal and vertical lines with known intersections. Verify grid is correctly detected. Test multi-grid pages. | Status: not_done
- [ ] **Unit test: text cluster detection** — Provide mock text elements with known x/y coordinates forming a table pattern. Verify column alignment detection, row grouping, and separation from prose text. | Status: not_done
- [ ] **Unit test: confidence scoring** — Provide tables with varying quality signals (complete grids, inconsistent columns, few rows, many empty cells). Verify scores fall in expected ranges. | Status: not_done

---

## Phase 4: Lattice Mode Extraction (`src/extract/lattice.ts`, `src/extract/grid.ts`)

- [ ] **Implement intersection detection (grid.ts)** — Find all points where a horizontal line crosses a vertical line. An intersection exists when V.x is within H.x-range and H.y is within V.y-range. Deduplicate intersections within snap tolerance. | Status: not_done
- [ ] **Implement grid construction (grid.ts)** — From deduplicated intersections, collect unique sorted x-coordinates (column boundaries) and unique sorted y-coordinates (row boundaries). Build a grid of cells where cell (r, c) occupies the rectangle from (x_c, y_r) to (x_{c+1}, y_{r+1}). Verify grid completeness by checking that all four boundary lines exist for each cell. | Status: not_done
- [ ] **Implement text-to-cell assignment for lattice** — For each text element, find the column c where x_c <= text.x < x_{c+1} and row r where y_r <= text.y < y_{r+1}. Assign text to cell (r, c). Concatenate multiple text elements in the same cell by y-coordinate then x-coordinate. Join lines with space or newline depending on `preserveCellNewlines`. Ignore text outside all grid cells. | Status: not_done
- [ ] **Implement lattice extraction pipeline (lattice.ts)** — Orchestrate the full lattice extraction for a detected table region: extract vector graphics, find h/v lines, find intersections, build grid, assign text to cells, detect headers, detect merged cells. Return an `ExtractedTable` object. | Status: not_done
- [ ] **Unit test: intersection detection** — Provide known horizontal and vertical lines. Verify all intersections are found. Test deduplication of nearby intersections. | Status: not_done
- [ ] **Unit test: grid construction** — Provide intersections forming known grids (3x3, 5x2, etc.). Verify correct row count, column count, and cell bounding boxes. | Status: not_done
- [ ] **Unit test: text-to-cell assignment (lattice)** — Provide a known grid and mock text elements at specific positions. Verify each element is assigned to the correct cell. Test multi-line cells and text outside the grid. | Status: not_done
- [ ] **Unit test: lattice extraction pipeline** — Provide a complete mock dataset (operator list with lines + text content). Verify end-to-end extraction produces correct headers, rows, and cell structure. | Status: not_done

---

## Phase 5: Header Detection (`src/extract/headers.ts`)

- [ ] **Implement font weight header detection** — Compare font names in the first row vs subsequent rows. If first row uses bold fonts (name contains "Bold", "Bd", "Heavy", "Semibold", "Medium") and subsequent rows do not, first row is the header. | Status: not_done
- [ ] **Implement font size header detection** — If first row's font size is >= 1.1x the font size of subsequent rows, first row is likely a header. | Status: not_done
- [ ] **Implement position-based header detection** — If a horizontal ruled line exists between the first and second rows but not between subsequent rows, first row is the header. Applies in lattice mode and tables with header-separator lines. | Status: not_done
- [ ] **Implement background color header detection** — Detect filled rectangles from the PDF operator stream that span the full width of the first row with a color different from the page background. If found, first row is likely a header. | Status: not_done
- [ ] **Implement content-based header detection** — If first row contains non-numeric, descriptive text (e.g., "Product", "Revenue") and subsequent rows contain numeric/mixed content, first row is the header. Use as a weak confirming signal only. | Status: not_done
- [ ] **Implement configurable header rows** — Support `headerRows` option: 0 (no header, generate synthetic "Column 1", "Column 2", ...), 1 (first row is header), 2+ (multiple header rows, flatten by concatenating parent/child text: "Q1 Revenue", "Q1 Cost"), 'auto' (use heuristic detection). | Status: not_done
- [ ] **Implement multi-level header flattening** — When multiple header rows are detected, flatten into a single header array by concatenating parent header text with child header text. Handle cases where parent headers span multiple child columns. | Status: not_done
- [ ] **Unit test: header detection** — Test bold first row, larger font first row, underlined first row, background color, content-based detection, ambiguous cases, explicit headerRows=0/1/2, and multi-level header flattening. | Status: not_done

---

## Phase 6: Merged Cell Detection (`src/extract/merged-cells.ts`)

- [ ] **Implement lattice mode colspan detection** — Detect horizontal merges from missing vertical grid lines between adjacent cells in the same row. Handle transitive merges (span extends across multiple missing lines). | Status: not_done
- [ ] **Implement lattice mode rowspan detection** — Detect vertical merges from missing horizontal grid lines between adjacent cells in the same column. Handle transitive merges. | Status: not_done
- [ ] **Implement combined merge detection (lattice)** — Detect cells that span both rows and columns simultaneously. Find the maximal rectangle of cells connected by missing internal grid lines. | Status: not_done
- [ ] **Implement stream mode colspan detection** — Detect text elements whose width spans from column c's left boundary beyond column c+1's right boundary. Detect text centered between multiple column boundaries. | Status: not_done
- [ ] **Implement stream mode rowspan detection** — Detect vertical merges where cell (r, c) has text and cell (r+1, c) is empty, suggesting a spanning header. Mark as weak signal; works best with explicit `headerRows` option. | Status: not_done
- [ ] **Implement flattened output for merged cells** — In the `headers` and `rows` arrays, duplicate merged cell text into every spanned position. In the `cells` array, set `rowSpan` and `colSpan` metadata on the origin cell and reference the origin from spanned positions. | Status: not_done
- [ ] **Unit test: merged cell detection** — Test lattice colspan, rowspan, and combined merges with known grids with missing lines. Test stream mode merges with spanning text. Verify flattened output duplicates values correctly. | Status: not_done

---

## Phase 7: Stream Mode Extraction (`src/extract/stream.ts`, `src/extract/columns.ts`, `src/extract/rows.ts`)

- [ ] **Implement x-coordinate collection (columns.ts)** — For all text elements in a table region, collect three sets: left edges (x), right edges (x + width), and centers (x + width / 2). | Status: not_done
- [ ] **Implement gap-based x-coordinate clustering (columns.ts)** — Sort x-values, walk through sorted list, start new cluster when gap exceeds threshold. Gap threshold: `max(averageCharWidth * 3, medianWordSpace * 2, 10 points)`. Each cluster's representative is the median of all values. | Status: not_done
- [ ] **Implement column boundary placement (columns.ts)** — Place boundaries at midpoints between adjacent cluster representatives: `(cluster[i].maxX + cluster[i+1].minX) / 2`. | Status: not_done
- [ ] **Implement alignment selection (columns.ts)** — Run clustering on left edges, right edges, and center x-coordinates. For each candidate, compute column count and boundary variance across rows. Select the candidate with lowest variance and most rows agreeing on column count. Support per-column alignment for mixed-alignment tables. | Status: not_done
- [ ] **Implement column validation (columns.ts)** — Require valid columns to contain text elements from at least 50% of detected rows. Discard single-column detections (reclassify as prose). Handle variable column widths and headers wider than data. | Status: not_done
- [ ] **Implement y-coordinate collection (rows.ts)** — For all text elements in the table region, collect the y-coordinate (baseline) of each element. | Status: not_done
- [ ] **Implement y-coordinate clustering (rows.ts)** — Sort y-values in reading order (normalize bottom-up PDF coordinates to top-to-bottom). Start new cluster when gap exceeds `max(medianFontSize * 0.6, 5 points)`. | Status: not_done
- [ ] **Implement multi-line row detection (rows.ts)** — After initial clustering, examine consecutive rows. Merge two adjacent y-clusters if: gap < 1.5x line height, at least one column has text in both clusters, other columns have text in only one cluster. Merged text is concatenated with space or newline per `preserveCellNewlines`. | Status: not_done
- [ ] **Implement text-to-cell assignment for stream mode** — For each text element at (x, y), find column c whose x-range contains x (or nearest column by distance) and row r whose y-range contains y. Assign text to cell (r, c). Concatenate with spaces, ordered by y then x. Handle text falling in column gaps by assigning to nearest column. | Status: not_done
- [ ] **Implement stream extraction pipeline (stream.ts)** — Orchestrate full stream extraction: extract text elements, detect columns, detect rows, handle multi-line cells, assign text to cells, detect headers, detect merged cells. Return `ExtractedTable` object. | Status: not_done
- [ ] **Unit test: column detection** — Test left-aligned, right-aligned, and center-aligned column detection. Test mixed alignment. Test variable column widths. Test single-column rejection. Test column validation (50% row threshold). | Status: not_done
- [ ] **Unit test: row detection** — Test y-coordinate clustering. Test multi-line row merging. Test bottom-up coordinate normalization. | Status: not_done
- [ ] **Unit test: stream extraction pipeline** — Provide mock text elements forming known borderless tables. Verify end-to-end extraction produces correct headers, rows, and cell structure. | Status: not_done

---

## Phase 8: Main Extractor Orchestrator (`src/extractor.ts`)

- [ ] **Implement input type detection** — Detect whether input is a file path (string ending in .pdf/.png/.jpg/.jpeg/.tiff/.tif), a Buffer, or a URL (string starting with http/https). Route to PDF or image extraction pipeline accordingly. | Status: not_done
- [ ] **Implement auto mode selection** — For each page/region, check if lattice mode detected grid lines. If yes, use lattice mode. If no, use stream mode. Per SPEC: auto mode examines each page and selects mode per table. | Status: not_done
- [ ] **Implement page selection** — Support `pages` option as `number[]` (specific pages, 1-based) or `{ start: number; end: number }` (range). Default: all pages. | Status: not_done
- [ ] **Implement `extractTables` function** — Main entry point: accept `TableInput` and `ExtractOptions`, detect input type, load document, iterate pages (respecting page selection), run detection, run extraction (lattice/stream/auto per table), apply multi-page merge if enabled, return `ExtractedTable[]`. | Status: not_done
- [ ] **Implement `extractTablesFromPDF` function** — PDF-specific entry point with PDF-specific options (password, mode, lattice/stream options). | Status: not_done
- [ ] **Implement `extractTablesFromImage` function** — Image-specific entry point with OCR options. | Status: not_done
- [ ] **Implement `detectTableRegions` function** — Detection-only entry point: run detection pipeline, return `TableRegion[]` without extracting cell content. Uses lower default minConfidence (0.3). | Status: not_done
- [ ] **Implement `createExtractor` factory function** — Create a `TableExtractor` instance with preset options. Instance methods (`extractTables`, `extractTablesFromPDF`, `extractTablesFromImage`, `detectTableRegions`) merge preset options with per-call overrides. | Status: not_done
- [ ] **Implement AbortSignal support** — Check `options.signal` before and during extraction. Throw `AbortError` if signal is aborted. Pass signal through to pdfjs-dist operations where applicable. | Status: not_done
- [ ] **Implement URL input fetching** — When input is a URL string, fetch the document over HTTP/HTTPS and pass the resulting Buffer to the appropriate extraction pipeline. | Status: not_done
- [ ] **Implement extraction timing** — Track elapsed time for each table extraction and populate `metadata.extractionMs`. | Status: not_done

---

## Phase 9: Multi-Page Table Merging (`src/extract/multi-page.ts`)

- [ ] **Implement merge candidate detection** — A table at the bottom of page N and a table at the top of page N+1 are merge candidates if: same column count, column boundary x-coordinates match within tolerance, top table does not end with a termination signal (thick bottom border, "Total" row), bottom table does not start with a new header. | Status: not_done
- [ ] **Implement duplicate header removal** — If the table on page N+1 starts with a header row whose content matches the header of the table on page N, remove the duplicate header row. | Status: not_done
- [ ] **Implement table merging** — Merge rows from the continuation table into the original table. Update `metadata.multiPage = true`, `metadata.pageRange = [N, N+1]`, and `metadata.rowCount`. | Status: not_done
- [ ] **Unit test: multi-page merging** — Test merge candidates with matching/mismatching column counts. Test duplicate header removal. Test termination signal detection ("Total" row, thick bottom border). | Status: not_done

---

## Phase 10: Output Formatters (`src/output/`)

- [ ] **Implement CSV export (csv.ts)** — RFC 4180 compliant CSV. Header row first. Quote fields containing commas, double quotes, or newlines. Escape internal double quotes by doubling (`""`). `table.toCSV()` returns the CSV string. | Status: not_done
- [ ] **Implement Markdown export (markdown.ts)** — GFM pipe table format. Escape pipe characters in cell content (`\|`). Separator row uses `---` per column. Support optional `alignment: true` parameter to add alignment markers (`:---`, `:---:`, `---:`). `table.toMarkdown()` returns the markdown string. | Status: not_done
- [ ] **Implement HTML export (html.ts)** — Generate complete `<table>` element with `<thead>`, `<tbody>`, `<tr>`, `<th>`, `<td>`. Include `rowspan` and `colspan` attributes for merged cells. Escape HTML entities in cell content (`&`, `<`, `>`, `"`, `'`). `table.toHTML()` returns the HTML string. | Status: not_done
- [ ] **Implement JSON export (json.ts)** — Return `{ headers: string[]; rows: string[][] }` plain object. This is the minimal portable representation. `table.toJSON()` returns the object. | Status: not_done
- [ ] **Implement ExtractedTable class** — Create the `ExtractedTable` class that holds `index`, `headers`, `rows`, `cells`, `metadata` and provides `toCSV()`, `toMarkdown()`, `toHTML()`, `toJSON()` methods delegating to the output formatters. | Status: not_done
- [ ] **Unit test: CSV export** — Test basic table, fields with commas, fields with double quotes, fields with newlines, empty cells, single-column table. | Status: not_done
- [ ] **Unit test: Markdown export** — Test basic table, pipe characters in content, alignment mode, single-column table. | Status: not_done
- [ ] **Unit test: HTML export** — Test basic table, merged cells (rowspan/colspan), HTML entity escaping, empty table. | Status: not_done

---

## Phase 11: Image-Based Extraction (`src/image/`)

- [ ] **Implement scanned page detection (image-extractor.ts)** — Classify a PDF page as scanned if `page.getTextContent()` returns fewer than 10 text elements AND the page contains image operators covering more than 80% of the page area. | Status: not_done
- [ ] **Implement PDF page renderer (renderer.ts)** — Render a PDF page to a PNG Buffer at configurable DPI (default: 300) using `pdfjs-dist` rendering with the `canvas` npm package. | Status: not_done
- [ ] **Implement tesseract.js OCR integration (ocr.ts)** — Accept an image Buffer, run OCR via `tesseract.js` with configurable language and confidence threshold. Convert OCR results (text with bounding boxes) into the internal `TextElement` format compatible with stream mode. Use wider clustering tolerances (1.5x normal) for OCR-sourced text. | Status: not_done
- [ ] **Implement standalone image input support** — Detect image file inputs (JPEG, PNG, TIFF by extension or magic bytes). For image inputs, skip PDF parsing and go directly to OCR pipeline. | Status: not_done
- [ ] **Implement image extraction pipeline (image-extractor.ts)** — Orchestrate image-based extraction: detect scanned page or accept image input, render page to image if needed, run OCR, apply stream mode on OCR output, return `ExtractedTable[]` with `metadata.ocrApplied = true` and `metadata.mode = 'ocr'`. | Status: not_done
- [ ] **Implement graceful peer dependency check** — When OCR is requested but `tesseract.js` or `canvas` is not installed, throw a clear error message explaining which peer dependency is missing and how to install it. | Status: not_done
- [ ] **Unit test: scanned page detection** — Test with mock pages having few text elements and large image operators (scanned) vs pages with many text elements (digital). | Status: not_done

---

## Phase 12: Vision LLM Integration (`src/image/llm.ts`)

- [ ] **Implement LLM extraction function** — Accept an image Buffer and the caller-provided `extract` function from `LLMOptions`. Construct the extraction prompt instructing the LLM to identify all tables and return JSON arrays with headers and rows. Parse the response into `ExtractedTable` objects with `metadata.mode = 'llm'`. | Status: not_done
- [ ] **Implement fallback mode logic** — When `llm.mode = 'fallback'`, invoke the LLM only when heuristic extraction produces tables with confidence below `llm.fallbackThreshold` (default: 0.4). When `llm.mode = 'always'`, always use LLM for image-based pages. | Status: not_done
- [ ] **Implement LLM response validation** — Validate the LLM response structure: must be an array of objects with `headers` (string[]) and `rows` (string[][]). Handle malformed responses gracefully (log warning, fall back to heuristic result). | Status: not_done
- [ ] **Unit test: LLM integration** — Test with a mock `extract` function returning known table data. Test fallback mode trigger conditions. Test malformed response handling. | Status: not_done

---

## Phase 13: CLI (`src/cli.ts`)

- [ ] **Implement CLI argument parsing** — Use Node.js built-in `util.parseArgs` (no external CLI framework). Parse all flags from SPEC Section 16: `--mode`, `--pages`, `--min-confidence`, `--header-rows`, `--no-merge`, `--password`, `--ocr`, `--ocr-language`, `--ocr-dpi`, `--format`, `--output`, `--pretty`, `--table`, `--detect-only`, `--version`, `--help`. Parse positional `<input>` argument. | Status: not_done
- [ ] **Implement --pages parsing** — Parse comma-separated pages (`1,2,5`) and ranges (`1-10`) into the `pages` option format (`number[]` or `{ start, end }`). | Status: not_done
- [ ] **Implement --help output** — Print usage information matching SPEC Section 16 format with all commands, flags, and descriptions. | Status: not_done
- [ ] **Implement --version output** — Read version from package.json and print it. | Status: not_done
- [ ] **Implement CLI extraction flow** — Read input from positional argument (file path or URL). Build `ExtractOptions` from parsed flags. Call `extractTables` or `detectTableRegions` (if `--detect-only`). | Status: not_done
- [ ] **Implement output formatting** — Format results based on `--format` flag: json (default, with optional `--pretty` for indentation), csv, markdown, html. Support `--table N` to output only the Nth table. | Status: not_done
- [ ] **Implement file output** — When `--output <path>` is provided, write output to the specified file instead of stdout. | Status: not_done
- [ ] **Implement exit codes** — Exit 0 on success (tables found), 1 when no tables found, 2 on input error (file not found, unsupported format, invalid options), 3 on extraction error (fatal error during processing). | Status: not_done
- [ ] **Add hashbang to cli.ts** — Add `#!/usr/bin/env node` at the top of the compiled `cli.js` output for direct invocation. | Status: not_done
- [ ] **Integration test: CLI** — Test CLI with various flags and inputs. Test JSON, CSV, markdown, HTML output. Test `--detect-only`. Test error exit codes. Test `--output` file writing. Test `--table` selection. | Status: not_done

---

## Phase 14: Error Handling and Edge Cases

- [ ] **Handle empty PDFs** — If a PDF has zero pages, return an empty `ExtractedTable[]` array without error. | Status: not_done
- [ ] **Handle pages with no tables** — If detection finds no tables on a page, skip it gracefully. If no tables are found in the entire document, return an empty array. CLI exits with code 1. | Status: not_done
- [ ] **Handle encrypted PDFs** — Pass `password` option to `pdfjs-dist`'s `getDocument()`. If the password is wrong or missing for an encrypted PDF, throw a descriptive error. | Status: not_done
- [ ] **Handle unsupported file formats** — If the input file is not a PDF or supported image format, throw a descriptive error (exit code 2 in CLI). | Status: not_done
- [ ] **Handle file not found** — If the input file path does not exist, throw a descriptive error (exit code 2 in CLI). | Status: not_done
- [ ] **Handle URL fetch failures** — If fetching a URL fails (network error, 404, etc.), throw a descriptive error. | Status: not_done
- [ ] **Handle corrupt PDFs** — If `pdfjs-dist` throws during PDF loading or page processing, catch and re-throw with a descriptive error message (exit code 3 in CLI). | Status: not_done
- [ ] **Handle single-row tables** — A detected region with only one row (the header) and no data rows is still a valid table. Return it with `rows: []` and `metadata.rowCount: 0`. | Status: not_done
- [ ] **Handle tables with all empty cells** — If all detected cells are empty, assign a low confidence score. Include in results if above minConfidence, skip otherwise. | Status: not_done
- [ ] **Handle deterministic output** — Ensure the same input with the same options always produces the same extraction result. No randomness in heuristic analysis, no network-dependent behavior during extraction (except URL fetch and LLM). | Status: not_done

---

## Phase 15: Integration Tests

- [ ] **Create lattice table PDF fixture** — Create or obtain a PDF containing a ruled table with visible grid lines, headers, and data rows. Place in `test/fixtures/lattice-table.pdf`. | Status: not_done
- [ ] **Create stream table PDF fixture** — Create or obtain a PDF containing a borderless table (no grid lines). Place in `test/fixtures/stream-table.pdf`. | Status: not_done
- [ ] **Create multi-table PDF fixture** — Create or obtain a PDF page containing two separate tables. Place in `test/fixtures/multi-table.pdf`. | Status: not_done
- [ ] **Create multi-page table PDF fixture** — Create or obtain a PDF with a table spanning two pages. Place in `test/fixtures/multi-page-table.pdf`. | Status: not_done
- [ ] **Create merged cells PDF fixture** — Create or obtain a PDF with tables containing colspan and rowspan cells. Place in `test/fixtures/merged-cells.pdf`. | Status: not_done
- [ ] **Create scanned page PDF fixture** — Create or obtain a scanned PDF (image-only page) containing a table. Place in `test/fixtures/scanned-page.pdf`. | Status: not_done
- [ ] **Create mixed mode PDF fixture** — Create or obtain a PDF with some pages having ruled tables and others having borderless tables. | Status: not_done
- [ ] **Integration test: lattice extraction (extract-lattice.test.ts)** — Extract tables from `lattice-table.pdf` using lattice mode. Verify all cells are correctly extracted, headers are identified, cell content matches expected values, and confidence is high (0.85+). | Status: not_done
- [ ] **Integration test: stream extraction (extract-stream.test.ts)** — Extract tables from `stream-table.pdf` using stream mode. Verify column detection accuracy, cell assignment, and header identification. | Status: not_done
- [ ] **Integration test: auto mode (extract-auto.test.ts)** — Extract tables from the mixed mode fixture using auto mode. Verify that lattice mode is selected for ruled pages and stream mode for borderless pages. | Status: not_done
- [ ] **Integration test: multi-table (extract-auto.test.ts)** — Extract from `multi-table.pdf`. Verify two separate tables are detected and correctly extracted. | Status: not_done
- [ ] **Integration test: multi-page merge (multi-page.test.ts)** — Extract from `multi-page-table.pdf` with `mergeMultiPageTables: true`. Verify tables are merged, duplicate headers removed, and metadata reflects multiPage status. | Status: not_done
- [ ] **Integration test: merged cells** — Extract from `merged-cells.pdf`. Verify colspan and rowspan are correctly detected. Verify flattened output duplicates merged values. | Status: not_done
- [ ] **Integration test: OCR extraction (ocr.test.ts)** — Extract from `scanned-page.pdf` with OCR enabled. Verify text is recognized and table structure is reconstructed. Test may be skipped if `tesseract.js` is not installed. | Status: not_done
- [ ] **Integration test: all export formats** — For a known extracted table, verify `toCSV()`, `toMarkdown()`, `toHTML()`, and `toJSON()` all produce correct output. | Status: not_done

---

## Phase 16: Performance and Optimization

- [ ] **Implement sequential page processing** — Process pages one at a time to control memory usage. Only one page's text content should be in memory at a time. Release page resources after extraction. | Status: not_done
- [ ] **Benchmark: single-page lattice table** — Measure extraction time for a single-page PDF with one lattice table. Target: < 200ms. | Status: not_done
- [ ] **Benchmark: single-page stream table** — Measure extraction time for a single-page PDF with one stream table. Target: < 300ms. | Status: not_done
- [ ] **Benchmark: 10-page PDF with 3 tables** — Measure end-to-end extraction time. Target: < 1 second. | Status: not_done
- [ ] **Benchmark: 100-page PDF with 15 tables** — Measure end-to-end extraction time. Target: < 5 seconds. | Status: not_done
- [ ] **Benchmark: memory usage** — Verify memory does not scale linearly with total page count for large documents. Monitor peak memory for OCR rendering (~25MB for letter-size at 300 DPI). | Status: not_done

---

## Phase 17: Documentation and Publishing

- [ ] **Write README.md** — Include: package description, installation instructions, quick start examples, API reference for all exported functions and types, CLI usage with all flags, configuration options table, output format examples, integration examples (with docling-node-ts, table-chunk, ai-file-router, rag-prompt-builder), peer dependency notes for OCR/canvas, performance characteristics, and license. | Status: not_done
- [ ] **Add JSDoc comments to all public exports** — Document `extractTables`, `extractTablesFromPDF`, `extractTablesFromImage`, `detectTableRegions`, `createExtractor`, and all public types/interfaces with JSDoc including `@param`, `@returns`, `@example`, and `@throws` tags. | Status: not_done
- [ ] **Version bump** — Bump version in `package.json` according to semver for the release (0.1.0 for initial development release, 1.0.0 for production release). | Status: not_done
- [ ] **Verify npm publish readiness** — Confirm `"files": ["dist"]` in package.json, `"main": "dist/index.js"`, `"types": "dist/index.d.ts"`, `"prepublishOnly": "npm run build"`. Run `npm pack --dry-run` and verify only intended files are included. | Status: not_done
- [ ] **Run full test suite** — Execute `npm run test` and verify all unit and integration tests pass. | Status: not_done
- [ ] **Run lint** — Execute `npm run lint` and fix any issues. | Status: not_done
- [ ] **Run build** — Execute `npm run build` and verify TypeScript compiles without errors. Verify `dist/` contains all expected .js and .d.ts files. | Status: not_done
- [ ] **Benchmark against Camelot/Tabula** — For each PDF fixture, extract tables using Camelot and Tabula for comparison. Document accuracy differences. Goal: equivalent data extraction accuracy for standard test cases. | Status: not_done
