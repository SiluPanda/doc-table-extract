# doc-table-extract -- Specification

## 1. Overview

`doc-table-extract` is a pure JavaScript library that extracts tables from PDF documents and images as structured JSON arrays. It accepts a file path, Buffer, or URL, detects tables on each page using spatial analysis of text element positions, reconstructs table structure (rows, columns, headers, merged cells), and returns an array of `ExtractedTable` objects containing the headers, data rows, cell-level metadata, and export methods for JSON, CSV, markdown, and HTML. No Python runtime, no Java dependency, no cloud API, no external service is required -- the entire extraction pipeline runs locally in Node.js.

The gap this package fills is the single most requested capability in JavaScript document processing. Table extraction from PDFs is consistently ranked among the top five unsolved problems in document AI, and every existing solution is either Python-only or cloud-only. The Python ecosystem has Camelot (lattice and stream extraction modes), Tabula (Java-based with a Python wrapper), pdfplumber (text-position analysis), and commercial APIs (AWS Textract, Google Document AI, Azure AI Document Intelligence). The JavaScript ecosystem has literally nothing. `pdf-parse` extracts raw text with no awareness of table structure -- a financial statement becomes an unreadable jumble of interleaved numbers and labels. `pdfjs-dist` provides positioned text elements but no table detection or structure recognition -- the developer must write their own spatial analysis, column detection, row grouping, header identification, and cell assignment on top of it. `tabula-js` wraps Java's Tabula binary and requires a full JVM installation as a deployment dependency. There is no pure JavaScript library that takes a PDF and returns structured tables.

The core technical challenge is that PDFs do not contain tables. The PDF specification has no "table" element, no row delimiter, no column boundary, no header marker, and no cell container. A table in a PDF is an optical illusion: it is a collection of text characters positioned at specific x,y coordinates on a page, optionally accompanied by drawn lines. What a human perceives as a three-column, ten-row financial table is stored internally as dozens of independent text fragments scattered across the page, each with a position, font, and size. Reconstructing the table structure from these fragments is fundamentally a spatial analysis problem: the library must find clusters of text elements that align in grid-like patterns, infer column boundaries from x-coordinate alignment, infer row boundaries from y-coordinate alignment, assign each text fragment to a cell, detect which row is the header, identify cells that span multiple rows or columns, and handle text that wraps within a cell. This is hard. It is why every existing solution uses either sophisticated heuristics (Camelot, pdfplumber) or machine learning models (Textract, Document AI).

`doc-table-extract` implements two extraction modes that mirror the industry-standard approaches established by Camelot. **Lattice mode** handles tables with visible grid lines (ruled tables) by detecting horizontal and vertical lines from the PDF's vector graphics stream, finding their intersections to form a grid, and mapping text elements into the grid cells. **Stream mode** handles tables without visible grid lines (borderless tables) by analyzing text element positions to detect column boundaries from x-coordinate clustering and row boundaries from y-coordinate clustering. Both modes produce the same output type: an array of `ExtractedTable` objects with headers, rows, cell metadata, bounding box, confidence score, and page number.

For scanned PDFs and images, the library integrates with `tesseract.js` for OCR to obtain text with positions, then applies stream-mode analysis on the OCR output. For cases where heuristic extraction is insufficient, an optional vision LLM integration sends the rendered page image to a multimodal model (GPT-4o, Claude Vision) for AI-powered table extraction.

The package provides both a TypeScript/JavaScript API for programmatic use and a CLI for extracting tables from the terminal. The API returns structured `ExtractedTable` objects with type-safe access to headers, rows, cells, and metadata. The CLI reads from a file path or URL and writes extracted tables to stdout or files as JSON, CSV, or markdown.

---

## 2. Goals and Non-Goals

### Goals

- Provide a primary function (`extractTables`) that accepts a document input (PDF file path, image file path, Buffer, or URL), detects all tables in the document, extracts their structure, and returns an `ExtractedTable[]` array.
- Provide format-specific functions (`extractTablesFromPDF`, `extractTablesFromImage`) for callers who know the input type and want format-specific options.
- Provide a detection-only function (`detectTableRegions`) that locates table regions on each page without extracting cell content, for callers who need bounding boxes or want to render table regions visually.
- Provide a factory function (`createExtractor`) that creates a configured extractor instance with preset options, avoiding repeated option parsing across multiple extractions.
- Implement **lattice mode** extraction for tables with visible grid lines: detect ruled lines from PDF vector graphics, find intersections, build a grid, and map text to cells.
- Implement **stream mode** extraction for tables without visible grid lines: detect columns from text x-coordinate clustering, detect rows from y-coordinate clustering, and assign text elements to cells.
- Implement **auto mode** (default) that examines each page for ruled lines and selects lattice mode if a grid is found, stream mode otherwise.
- Detect and extract multiple tables per page.
- Detect table headers from font weight, font size, position, and configurable rules.
- Detect merged cells (colspan and rowspan) in both lattice and stream modes.
- Handle multi-line cells where text wraps within a single cell.
- Handle tables that span multiple pages by detecting continuation patterns and merging across page boundaries.
- Support page selection to extract tables from specific pages only.
- Provide multiple output formats: JSON (structured `ExtractedTable` objects), CSV, GFM markdown tables, and HTML `<table>` elements.
- Provide a CLI (`doc-table-extract`) that extracts tables from a file or URL and writes output to stdout or a file.
- Support scanned PDFs and images via optional OCR integration with `tesseract.js`.
- Support optional vision LLM integration for AI-powered table extraction from rendered page images.
- Produce deterministic output: the same input with the same options always produces the same extraction result. No network access during extraction (except for URL input fetching and optional LLM integration), no non-determinism in heuristic analysis.
- Target Node.js 18 and above.

### Non-Goals

- **Not a full document converter.** This package extracts tables and only tables. It does not convert PDFs to markdown, extract headings, reconstruct reading order, or produce a full document representation. For full document conversion, use `docling-node-ts`. The relationship is complementary: `docling-node-ts` handles the full document; `doc-table-extract` provides a specialized, higher-accuracy table extraction engine that `docling-node-ts` can delegate to for the hardest sub-problem.
- **Not a table chunker.** This package extracts tables from documents into structured objects. Splitting those objects into embedding-optimized chunks for RAG is the responsibility of `table-chunk`. The canonical pipeline is: `doc-table-extract` (extract) then `table-chunk` (chunk) then `embed-cache` (embed).
- **Not a PDF renderer.** This package does not render PDFs to images for display. It renders pages to images internally only when OCR or vision LLM integration is needed. For PDF rendering, use `pdfjs-dist` directly.
- **Not an OCR engine.** OCR capability is delegated entirely to `tesseract.js` as an optional peer dependency. This package does not implement character recognition. For production OCR, consider dedicated OCR services.
- **Not a machine learning framework.** Table detection and structure recognition use heuristic spatial analysis, not trained models. For ML-powered table detection (comparable to Google Document AI or AWS Textract accuracy), use those services. This package targets the 80-90% of tables that are well-structured enough for heuristic analysis.
- **Not a spreadsheet parser.** This package extracts tables from PDFs and images. It does not parse Excel (`.xlsx`), Google Sheets, or other spreadsheet formats. For spreadsheet parsing, use `xlsx` or `exceljs`.
- **Not a web scraper.** This package does not fetch web pages, parse HTML tables, or handle dynamic content. For HTML table extraction, use `cheerio` or `table-chunk` (which parses HTML tables from markup). This package operates on PDFs and images.
- **Not a LangChain integration.** This package is framework-independent. It returns typed objects. Wrapping the output into LangChain `Document` objects is trivial and left to the caller.

---

## 3. Target Users and Use Cases

### RAG Pipeline Builders

Developers constructing retrieval-augmented generation pipelines who need to extract structured table data from PDF documents for embedding and retrieval. Tables are the single most problematic content type in RAG: a standard text splitter destroys row-column relationships, producing chunks where numerical values are separated from their column headers. The pipeline is: `doc-table-extract` extracts tables as structured objects, `table-chunk` chunks them with header preservation, `embed-cache` embeds the chunks. A typical integration: `const tables = await extractTables('./10-K-filing.pdf'); for (const table of tables) { const chunks = chunkTable(table); }`.

### Financial Report Processing

Analysts and data teams processing financial statements, SEC filings, earnings reports, and regulatory documents. These PDFs contain dense numerical tables -- income statements, balance sheets, cash flow statements -- where every cell value must be correctly associated with its row label and column header. Extracting "Revenue: $4.2B" requires knowing that "$4.2B" is in the "Q3 2025" column and the "Revenue" row. `doc-table-extract` preserves this structure. A financial data pipeline: `const tables = await extractTables('./10-K.pdf', { pages: [45, 46, 47] }); const incomeStatement = tables.find(t => t.headers.includes('Revenue'));`.

### Invoice and Receipt Processing

Backend engineers building invoice processing systems that extract line items, totals, tax amounts, and payment terms from PDF invoices. Invoices typically contain ruled tables (lattice mode) with item descriptions, quantities, unit prices, and totals. `doc-table-extract`'s lattice mode handles these reliably because the grid lines provide unambiguous cell boundaries.

### Academic Paper Data Extraction

Researchers extracting experimental results, comparison tables, and statistical data from academic papers. Papers typically use borderless tables (stream mode) with clean column alignment. The challenge is detecting tables among prose text and handling multi-line cells with long descriptions. `doc-table-extract`'s stream mode detects columns from text alignment and handles multi-line cells by grouping text elements within the same column-row intersection.

### Document Digitization

Organizations converting paper archives to structured data. Legacy documents are scanned to PDF (image-only pages), and tables in those scans must be extracted. `doc-table-extract` integrates with `tesseract.js` to OCR scanned pages, then applies stream-mode analysis on the recognized text positions to reconstruct table structure.

### Data Migration and ETL

Data engineers extracting tabular data from PDF reports generated by legacy systems. These reports are often the only export format available from old enterprise software. The extracted tables are loaded into databases, data warehouses, or modern analytics tools. `doc-table-extract` provides CSV and JSON export for direct database ingestion.

### Integration with npm-master Ecosystem

Developers using other packages in the npm-master monorepo. `doc-table-extract` is the specialized table extraction engine in the document processing pipeline: it extracts the structured tables that `table-chunk` chunks, that `docling-node-ts` can delegate to for its table detection stage, that `ai-file-router` can route PDF files to, and that `rag-prompt-builder` can format into LLM context. The relationship with `docling-node-ts` is particularly important: `docling-node-ts` performs broad document conversion (headings, paragraphs, lists, images, tables), while `doc-table-extract` provides deep, specialized table extraction with higher accuracy, more configuration options, and cell-level metadata that `docling-node-ts` does not expose.

---

## 4. Core Concepts

### Table Detection

Table detection is the process of finding table regions on a page. A PDF page may contain zero, one, or many tables interspersed with prose text, headings, images, and other content. Detection identifies candidate rectangular regions that likely contain tables, assigns each a confidence score, and passes them to the structure recognition stage. Detection is the first stage of the extraction pipeline and determines what gets extracted. A missed detection means a table is silently skipped. A false detection means prose text is incorrectly parsed as a table. Both failure modes are problematic; the heuristics are tuned to minimize false negatives (missed tables) at the cost of tolerating a small rate of false positives (which are filtered by the structure recognition stage).

### Table Structure Recognition

Table structure recognition is the process of reconstructing the internal grid of a detected table region: how many rows, how many columns, where the boundaries are, which text belongs to which cell, which row is the header, and which cells are merged. This is the core algorithmic challenge. Structure recognition takes a bounding box containing positioned text elements and produces a structured grid of cells with row indices, column indices, text content, and span information. The two modes (lattice and stream) use fundamentally different approaches to this problem.

### Lattice Mode

Lattice mode is the extraction strategy for tables with visible grid lines. "Lattice" refers to the grid pattern formed by horizontal and vertical ruled lines that delineate cell boundaries. In a PDF, these lines are vector graphics -- draw operations in the page's content stream that render horizontal or vertical line segments. Lattice mode detects these lines, finds their intersections, builds a grid from the intersections, defines cells as the rectangular regions bounded by grid lines, and maps text elements into cells. Lattice mode is highly reliable because the grid lines provide explicit cell boundaries. It is the preferred mode for invoices, financial reports, forms, and any table with visible borders.

### Stream Mode

Stream mode is the extraction strategy for tables without visible grid lines. "Stream" refers to the stream of text elements that must be analyzed for alignment patterns. Without grid lines, cell boundaries must be inferred entirely from text positioning: columns are detected from clusters of text elements with similar x-coordinates (left edges), and rows are detected from clusters with similar y-coordinates. Stream mode is inherently less reliable than lattice mode because text alignment is an indirect signal -- columns may have varying alignment (left, center, right), spacing between columns may vary, and distinguishing a table from aligned prose requires heuristic judgment. Stream mode handles academic paper tables, web-generated PDFs, and any borderless table.

### Cell

A cell is the atomic unit of a table: the intersection of one row and one column, containing text content. In the output, each cell is represented with its text value, row index, column index, and optional span information (colspan, rowspan). A cell may contain multi-line text if the original text wraps within the cell boundaries. Multi-line cell content is joined with spaces or preserved with newlines depending on configuration.

### Header

A header is a row (or set of rows) at the top of a table that labels the columns. Headers are semantically different from data rows: they describe what values mean rather than containing values themselves. Header detection uses multiple signals: font weight (bold text), font size (larger than body), position (first row, or first row after a ruled line), and configurable explicit specification. Correct header detection is critical because headers are repeated in every chunk when `table-chunk` splits the table for RAG.

### Merged Cell

A merged cell spans multiple rows, multiple columns, or both. In lattice mode, a merged cell occupies a grid region larger than one unit cell -- a cell spanning two columns covers the area between three vertical grid lines. In stream mode, a merged cell is text that extends across column boundaries or repeats across row boundaries. Merged cells are represented with `rowSpan` and `colSpan` values greater than 1. In the flattened output (headers/rows arrays), merged cell values are duplicated into each spanned position.

### Confidence Score

A confidence score (0.0 to 1.0) indicates how likely a detected region is to be an actual table. The score is computed from multiple signals: number of aligned rows and columns, consistency of column spacing, presence of grid lines, number of cells with content, and ratio of empty to filled cells. Higher confidence means the region more closely matches expected table patterns. The `minConfidence` option filters out low-confidence detections.

---

## 5. Table Detection

Table detection is the first stage of the extraction pipeline. It scans each page of the input document and identifies rectangular regions that contain tables. Detection operates independently from structure recognition: it finds where tables are, not what is inside them.

### 5.1 Ruled Line Detection

For lattice mode, the primary detection signal is the presence of horizontal and vertical lines drawn on the page. PDFs render lines using vector graphics operators in the page's content stream: `m` (moveto), `l` (lineto), `re` (rectangle), and `S`/`s`/`f`/`F` (stroke/fill) operators. The line detector processes the page's operator list from `pdfjs-dist` and extracts line segments.

**Algorithm:**

1. **Operator stream parsing**: Iterate through the page's operator list (obtained via `page.getOperatorList()` from `pdfjs-dist`). Identify stroke and fill operations that produce line segments. The relevant operators are:
   - `OPS.constructPath`: Contains path construction commands (moveto, lineto, curveto, rectangle).
   - `OPS.stroke` / `OPS.fill`: Render the constructed path.
   - `OPS.paintSolidColorImageMask`: Sometimes used for thin rectangles that act as lines.

2. **Line extraction**: From the path data, extract individual line segments as `{ x1, y1, x2, y2 }` coordinates. A line is classified as horizontal if `|y1 - y2| < tolerance` (default: 2 points) and its length exceeds a minimum threshold (default: 20 points). A line is classified as vertical if `|x1 - x2| < tolerance` and its height exceeds the minimum threshold. Diagonal lines and curves are ignored -- they are not table grid lines.

3. **Rectangle decomposition**: Rectangle operators (`re`) produce four line segments (top, bottom, left, right). Thin rectangles (width or height below 3 points) are treated as thick lines rather than cells.

4. **Line clustering**: Horizontal lines with similar y-coordinates (within tolerance) are grouped. Vertical lines with similar x-coordinates are grouped. This merges line segments that form a continuous grid line from multiple drawing operations.

5. **Grid detection**: A grid exists if there are at least 2 horizontal lines and at least 2 vertical lines that intersect, forming a rectangular lattice. The bounding box of the grid defines the table region.

6. **Multi-table detection**: If multiple non-overlapping grids are detected on a single page, each is reported as a separate table region.

### 5.2 Text Cluster Detection

For stream mode (and as a secondary signal for lattice mode), tables are detected from clusters of text elements arranged in grid-like patterns.

**Algorithm:**

1. **Text element collection**: Extract all text content items from the page via `page.getTextContent()` from `pdfjs-dist`. Each item has a string value, x-coordinate, y-coordinate, width, height, font name, and font size.

2. **Row candidate detection**: Group text elements by y-coordinate. Elements with y-coordinates within one line-height tolerance (computed from the median font size) are assigned to the same row group. A row group must contain at least 2 text elements separated by significant horizontal gaps.

3. **Column alignment analysis**: For each row group, collect the x-coordinates of text element left edges. Across all row groups on the page, identify x-coordinate values that recur in multiple rows. Recurring x-coordinates indicate column alignment. A minimum of 3 consecutive rows sharing the same column alignment pattern constitutes a table candidate.

4. **Table region bounding**: The bounding box of a text-cluster table is computed from the minimum and maximum x,y coordinates of all text elements that participate in the aligned grid, plus a small padding margin.

5. **Separation from prose**: Text clusters are distinguished from aligned prose by two criteria: (a) the cluster has at least 2 distinct columns (prose is single-column), and (b) the horizontal gap between columns is significantly larger than word spacing within a column (at least 2x the average word space).

### 5.3 Region Proposals and Filtering

Detection produces a set of candidate table regions. Each region has a bounding box (`{ x, y, width, height }` in page coordinates), a detection method (`lattice` or `text-cluster`), and a raw confidence score.

**Filtering rules:**

1. **Minimum size**: Regions smaller than a configurable threshold (default: 50x50 points) are discarded. A region this small cannot contain a meaningful table.
2. **Minimum cells**: Regions with fewer than 2 rows or 2 columns of detected content are discarded.
3. **Overlap resolution**: If two detected regions overlap by more than 50% of the smaller region's area, the one with higher confidence is kept and the other is discarded.
4. **Confidence threshold**: Regions with confidence below `minConfidence` (default: 0.5) are discarded.

### 5.4 Confidence Scoring

The confidence score for a detected table region is computed from multiple signals, normalized to a 0.0-1.0 range:

| Signal | Weight | Scoring |
|--------|--------|---------|
| Grid lines present | 0.30 | 1.0 if horizontal and vertical grid lines form a complete lattice, 0.0 if no lines |
| Column count consistency | 0.20 | 1.0 if all rows have the same column count, decreasing with variance |
| Row count | 0.15 | Scales from 0.3 (2 rows) to 1.0 (5+ rows). Tables with more rows are more confident. |
| Column spacing consistency | 0.15 | 1.0 if column boundaries are at consistent x-positions across rows, decreasing with deviation |
| Cell fill ratio | 0.10 | Fraction of cells that contain text (empty tables are less confident) |
| Font consistency | 0.10 | 1.0 if data cells use the same font, decreasing with font variation (tables typically use uniform fonts) |

The final score is the weighted sum of all signals. Lattice-detected tables typically score 0.85-1.0 because grid lines are a strong signal. Stream-detected tables typically score 0.5-0.8 depending on alignment consistency.

### 5.5 Multi-Table Pages

A single page may contain multiple tables. The detection stage identifies each table independently and returns them as separate `TableRegion` objects. Tables are ordered by their position on the page: top-to-bottom, then left-to-right for tables at the same vertical position.

Common multi-table patterns:

- **Stacked tables**: Two tables vertically separated by prose text or a heading. Detected as separate regions because the vertical gap between them breaks the row continuity.
- **Side-by-side tables**: Two tables horizontally adjacent. Detected as separate regions because the horizontal gap between them breaks the column continuity. Less common in PDFs.
- **Table with caption**: A table preceded by a caption line. The caption is not part of the table region -- it is prose text above the table's bounding box.

---

## 6. Lattice Mode

Lattice mode extracts tables that have visible grid lines (ruled tables). This is the most reliable extraction mode because grid lines provide explicit cell boundaries, eliminating the ambiguity of inferring structure from text positions alone. Lattice mode is the correct choice for invoices, financial reports, government forms, contracts, and any document where tables are drawn with visible borders.

### 6.1 Step 1: Extract Vector Graphics

The first step reads the PDF page's content stream and extracts all drawn lines and rectangles. This uses the operator list from `pdfjs-dist`'s `page.getOperatorList()`.

**Implementation:**

```
For each operator in operatorList:
  If operator is constructPath:
    Parse the path commands (moveto, lineto, rectangle)
    Record line segments and rectangles with coordinates
  If operator is stroke or fill:
    Finalize the pending path as a drawn element
    Apply the current transformation matrix to get page coordinates
```

Lines are stored as `{ x1, y1, x2, y2, lineWidth }` objects. Rectangles are decomposed into four line segments. The transformation matrix from the graphics state is applied to convert from user space to page space coordinates.

### 6.2 Step 2: Find Horizontal and Vertical Lines

Filter the extracted line segments to identify horizontal and vertical lines that could form table grid boundaries.

**Horizontal line criteria:**
- Absolute difference between y1 and y2 is less than the tolerance (default: 2 points).
- Length (`|x2 - x1|`) exceeds the minimum line length (default: 20 points).
- Line width is below a maximum (default: 5 points) -- thicker lines are borders or decorative elements, not grid lines.

**Vertical line criteria:**
- Absolute difference between x1 and x2 is less than the tolerance.
- Height (`|y2 - y1|`) exceeds the minimum line length.
- Line width is below the maximum.

**Snapping:** Lines whose y-coordinates (for horizontal) or x-coordinates (for vertical) are within the snap tolerance (default: 3 points) of each other are snapped to the same coordinate. This accounts for minor rendering imprecisions where grid lines are drawn at slightly different positions.

### 6.3 Step 3: Find Line Intersections

Find all points where a horizontal line crosses a vertical line. An intersection occurs when a horizontal line's y-coordinate falls within the vertical line's y-range, and the vertical line's x-coordinate falls within the horizontal line's x-range.

**Algorithm:**

```
intersections = []
For each horizontal line H:
  For each vertical line V:
    If V.x >= H.x1 and V.x <= H.x2:
      If H.y >= V.y1 and H.y <= V.y2:
        intersections.push({ x: V.x, y: H.y })
```

Intersections are deduplicated: points within the snap tolerance of each other are merged to a single intersection. The set of unique intersections defines the grid nodes.

### 6.4 Step 4: Build the Grid

From the intersections, construct a regular grid of cells.

**Algorithm:**

1. Collect all unique x-coordinates from intersections and sort them in ascending order. These are the column boundaries: `[x0, x1, x2, ..., xN]`.
2. Collect all unique y-coordinates from intersections and sort them. These are the row boundaries: `[y0, y1, y2, ..., yM]`.
3. The grid has `(N)` columns and `(M)` rows. Cell `(r, c)` occupies the rectangle from `(x_c, y_r)` to `(x_{c+1}, y_{r+1})`.
4. Verify grid completeness: for each cell, check that all four boundary lines exist (top, bottom, left, right). Missing boundary lines indicate an incomplete grid -- the cell may be part of a merged cell or the grid line detection missed a segment.

### 6.5 Step 5: Assign Text to Cells

Map each text element on the page to the grid cell that contains it.

**Algorithm:**

1. For each text element with position `(text.x, text.y)`:
   - Find the column `c` such that `x_c <= text.x < x_{c+1}`.
   - Find the row `r` such that `y_r <= text.y < y_{r+1}`.
   - Assign the text to cell `(r, c)`.
2. Multiple text elements assigned to the same cell are concatenated. Concatenation order is determined by y-coordinate (top to bottom) then x-coordinate (left to right). Text elements on different lines within the same cell are joined with a space (or newline, if `preserveCellNewlines` is enabled).
3. Text elements that fall outside all grid cells (in margins, in gaps between grid lines and page edges) are ignored -- they are not part of the table.

### 6.6 Step 6: Detect Headers

Identify which rows are header rows.

**Signals for header detection:**

1. **Font weight**: If the first row's text elements use a bold font (font name contains "Bold", "Bd", or "Heavy") and subsequent rows use a regular font, the first row is the header.
2. **Font size**: If the first row's font size is larger than subsequent rows, it is the header.
3. **Horizontal ruled line**: If a thicker or double horizontal line separates the first row from the rest, the first row is the header.
4. **Content analysis**: If the first row contains non-numeric text labels and subsequent rows contain numeric data, the first row is the header.
5. **Explicit configuration**: The `headerRows` option allows the caller to specify the exact number of header rows (default: auto-detect, typically 1).

When multiple header rows are detected (multi-level headers), they are flattened into a single header array by concatenating parent and child header text: a header "Q1" spanning columns "Revenue" and "Cost" becomes `["Q1 Revenue", "Q1 Cost"]`.

### 6.7 Step 7: Detect Merged Cells

Identify cells that span multiple rows or columns.

**In lattice mode**, merged cells are detected from missing internal grid lines:

1. For adjacent cells `(r, c)` and `(r, c+1)`: if the vertical line between them is absent (no line segment at `x_{c+1}` between `y_r` and `y_{r+1}`), the cells are merged horizontally. The merged cell has `colSpan = 2` (or more, if additional adjacent lines are also missing).
2. For adjacent cells `(r, c)` and `(r+1, c)`: if the horizontal line between them is absent, the cells are merged vertically. The merged cell has `rowSpan = 2` (or more).
3. Combined merges: a cell can span multiple rows and multiple columns simultaneously.

In the flattened output (`headers` and `rows` arrays), merged cell values are duplicated into every position they span. In the `cells` output, merged cells carry `rowSpan` and `colSpan` metadata and spanned positions reference the origin cell.

---

## 7. Stream Mode

Stream mode extracts tables that do not have visible grid lines (borderless tables). This mode is necessary for the majority of tables in academic papers, web-generated PDFs, many government reports, and any document where tables rely on whitespace alignment rather than drawn borders. Stream mode is inherently less reliable than lattice mode because it must infer cell boundaries from text positioning patterns rather than explicit grid lines.

### 7.1 Step 1: Extract Text Elements

Extract all text content items with their positions from the PDF page.

Each text element has:
- `text`: The string content.
- `x`: The x-coordinate of the left edge.
- `y`: The y-coordinate of the baseline.
- `width`: The rendered width of the text.
- `height`: The font size (approximately).
- `fontName`: The font identifier.
- `fontSize`: The computed font size from the transformation matrix.

Text elements are obtained from `pdfjs-dist`'s `page.getTextContent()`, which returns `TextItem` objects. The transformation matrix `[a, b, c, d, e, f]` encodes the position (`e` = x, `f` = y) and font size (`Math.sqrt(a*a + b*b)`).

### 7.2 Step 2: Detect Column Boundaries

Column detection is the most critical step in stream mode. It determines where one column ends and the next begins.

**Algorithm: Gap-Based X-Coordinate Clustering**

1. **Collect left edges**: For every text element in the table region, record the x-coordinate of the left edge. Also record the x-coordinate of the right edge (`x + width`).

2. **Build x-histogram**: Create a histogram of left-edge x-coordinates with a bin width of 2 points. Peaks in the histogram indicate column start positions.

3. **Detect gaps**: Sort the histogram peaks by x-coordinate. Gaps between adjacent peaks that exceed a threshold (default: the median word-space width multiplied by 3) indicate column boundaries. The column boundary is placed at the midpoint of each gap.

4. **Validate columns**: A valid column must contain text elements from at least 50% of the detected rows. A "column" that only appears in one row is likely a coincidental alignment, not a real column.

5. **Handle alignment variations**:
   - **Left-aligned columns**: Text left edges cluster at a consistent x-coordinate. This is the default case handled by the algorithm above.
   - **Right-aligned columns**: Text right edges cluster at a consistent x-coordinate (common for numeric columns). Detected by running the same clustering on right edges.
   - **Center-aligned columns**: Text center points cluster at a consistent x-coordinate. Detected by running clustering on center x-coordinates `(x + width/2)`.
   - The algorithm runs all three alignment analyses and selects the one that produces the most consistent column structure.

### 7.3 Step 3: Detect Row Boundaries

Row detection groups text elements into horizontal bands.

**Algorithm: Y-Coordinate Clustering**

1. **Collect baselines**: For every text element, record the y-coordinate (baseline position).

2. **Cluster y-coordinates**: Sort y-coordinates and identify clusters. Two text elements belong to the same row if their y-coordinates differ by less than the line-height tolerance (default: 1.2 times the median font size).

3. **Row boundaries**: Each cluster of y-coordinates defines a row. The row boundary is the y-range from the minimum to the maximum y-coordinate in the cluster, expanded by half a line height above and below.

4. **Multi-line row detection**: If two consecutive y-clusters are closer together than the typical inter-row gap (the gap is less than 0.5 times the median inter-cluster gap), they may be part of the same logical row (a multi-line cell). The multi-line detection merges these clusters if the text elements in both clusters align with the same column structure. See Section 7.5 for details.

### 7.4 Step 4: Assign Text to Cells

With column boundaries and row boundaries established, each text element is mapped to a (row, column) cell.

**Algorithm:**

1. For each text element at position `(x, y)`:
   - Find the column `c` whose x-range contains `x` (or whose boundary the text element's center falls within).
   - Find the row `r` whose y-range contains `y`.
   - Assign the text to cell `(r, c)`.
2. Multiple text elements assigned to the same cell are concatenated with spaces, ordered by y-coordinate then x-coordinate.
3. If a text element's x-coordinate falls between two column boundaries (in the gap), it is assigned to the nearest column by minimum horizontal distance.
4. If a text element spans two column boundaries (its width extends from one column into the next), it may indicate a merged cell. See Section 10 for merged cell detection in stream mode.

### 7.5 Step 5: Handle Multi-Line Cells

Text wrapping within a cell produces multiple text elements at different y-coordinates but within the same column. These must be grouped into a single cell value.

**Algorithm:**

1. Within each column, examine consecutive rows. If a "row" contains text in only one or two columns (while adjacent rows use all columns), the sparse row may be a continuation of the row above.
2. Merge candidate: Two consecutive y-clusters are merged into one logical row if:
   - The gap between them is less than 1.5 times the line height.
   - At least one column has text in both y-clusters (text wraps to a new line in that column).
   - Other columns have text in only one of the two y-clusters (the text in those columns does not wrap).
3. Merged text elements within a column are concatenated with a space (or newline if `preserveCellNewlines` is true).

### 7.6 Step 6: Detect Headers

Header detection in stream mode uses the same signals as lattice mode (Section 6.6), but without the grid-line signal:

1. **Font analysis**: Bold or larger font in the first row.
2. **Content analysis**: Non-numeric labels in the first row, numeric data in subsequent rows.
3. **Explicit configuration**: `headerRows` option.
4. **Underline detection**: A thin horizontal line below the first row (even in an otherwise borderless table) strongly signals a header separator.

### 7.7 When to Use Stream Mode

Stream mode is automatically selected by auto mode when no grid lines are detected on a page. It is the appropriate mode for:

- Academic papers with borderless result tables.
- Web-generated PDFs (wkhtmltopdf, Chrome print-to-PDF) where tables are rendered with CSS styling but no drawn borders in the PDF content stream.
- Government reports and technical documents with simple tabular layouts.
- Any table that relies on whitespace alignment rather than visible lines.

Stream mode is less reliable than lattice mode. Accuracy depends on how consistently the source document aligns text into columns. Tables with irregular spacing, inconsistent column widths, or merged cells without clear alignment cues may produce incorrect extraction results. The confidence score for stream-mode tables reflects this uncertainty.

---

## 8. Column Detection Algorithm

Column detection is the single most important algorithm in stream mode. If columns are detected incorrectly, every cell assignment is wrong. This section specifies the algorithm in detail.

### 8.1 X-Coordinate Collection

For all text elements in the candidate table region, collect three x-coordinate sets:

- **Left edges**: `x` for each text element.
- **Right edges**: `x + width` for each text element.
- **Centers**: `x + width / 2` for each text element.

### 8.2 Clustering

For each x-coordinate set, apply gap-based clustering:

1. Sort the x-values in ascending order.
2. Walk through the sorted list. If the gap between consecutive values exceeds the gap threshold, start a new cluster.
3. The gap threshold is computed as: `max(averageCharWidth * 3, medianWordSpace * 2, 10 points)`. The threshold must be large enough to ignore word spacing within a column but small enough to catch column gaps.
4. Each cluster's representative x-coordinate is the median of all values in the cluster.

### 8.3 Column Boundary Placement

Column boundaries are placed at the midpoints between adjacent cluster representatives:

```
boundary[i] = (clusterRepresentative[i].right + clusterRepresentative[i+1].left) / 2
```

Where `clusterRepresentative[i].right` is the maximum x-value in cluster `i` and `clusterRepresentative[i+1].left` is the minimum x-value in cluster `i+1`.

### 8.4 Alignment Selection

The three x-coordinate sets (left, right, center) produce three candidate column structures. The algorithm selects the one that produces the most consistent results:

1. For each candidate, compute the column count and the variance of column boundary positions across rows.
2. The candidate with the lowest boundary variance and the most rows agreeing on the column count is selected.
3. If a table has mixed alignment (some columns left-aligned, some right-aligned), a per-column alignment analysis is used: each column's alignment is independently determined from whether left, right, or center coordinates cluster most tightly.

### 8.5 Edge Cases

- **Single-column "tables"**: If only one column is detected, the region is not a table. It is reclassified as prose and excluded from results.
- **Variable column widths**: Columns may have different widths. The algorithm handles this naturally because it clusters x-coordinates rather than assuming equal-width columns.
- **Column headers wider than data**: A header cell may be wider than the data cells below it (e.g., "Product Description" header over short product names). The column boundary is determined by the data cells, and the header text is trimmed or wrapped.

---

## 9. Row Detection Algorithm

Row detection groups text elements into horizontal rows. This is simpler than column detection because y-coordinates in a well-structured table vary less than x-coordinates.

### 9.1 Y-Coordinate Collection

For all text elements in the candidate table region, collect the y-coordinate (baseline) of each element.

### 9.2 Clustering

Apply gap-based clustering on y-coordinates:

1. Sort y-values (in page order -- typically top-to-bottom, though PDF y-coordinates may be bottom-up depending on the coordinate system. The algorithm normalizes to top-to-bottom order).
2. Walk through sorted values. If the gap between consecutive values exceeds the row gap threshold, start a new cluster.
3. Row gap threshold: `max(medianFontSize * 0.6, 5 points)`. The threshold must be large enough to group text elements on the same line (which may have slightly different baselines due to subscripts, superscripts, or font variations) but small enough to separate distinct rows.

### 9.3 Multi-Line Row Detection

After initial row clustering, consecutive rows are examined for multi-line merging (see Section 7.5). The key heuristic: if merging two adjacent rows produces a row where every column has content, and keeping them separate produces rows where some columns are empty, merging is preferred.

### 9.4 Row Ordering

Rows are ordered by their y-coordinate in reading order (top-to-bottom). In PDFs with bottom-up coordinate systems, y-coordinates are inverted before ordering. The final row indices in the `ExtractedTable` output are 0-based from top to bottom.

---

## 10. Header Detection

Header detection identifies which rows at the top of a table are column labels rather than data. Correct header detection is critical for downstream use: `table-chunk` repeats headers in every chunk, RAG queries interpret values through their column headers, and CSV export uses the header row as column names.

### 10.1 Font Weight Analysis

The most reliable signal. Examine the font name of text elements in the first row compared to subsequent rows.

- If the first row's text elements use fonts whose names contain "Bold", "Bd", "Heavy", "Semibold", or "Medium", and subsequent rows use fonts without these markers, the first row is the header.
- Implementation: for each text element, resolve the font name from `pdfjs-dist`'s font metadata (`commonObjs`). Check for bold indicators in the font name string.

### 10.2 Font Size Analysis

If the first row's text elements have a larger font size than subsequent rows (ratio >= 1.1x), the first row is likely a header.

### 10.3 Position-Based Detection

If a horizontal ruled line (detected during the lattice mode line detection stage) appears between the first and second rows but not between subsequent rows, the first row is the header. This signal is applicable in lattice mode and in tables that have a header-separator line but no other grid lines.

### 10.4 Background Color Detection

Some tables shade the header row with a background color. Background colors are drawn as filled rectangles behind the text. If a filled rectangle with a color different from the page background spans the full width of the first row, the first row is likely a header. This signal is extracted from the PDF operator stream during vector graphics analysis.

### 10.5 Content-Based Analysis

If the first row contains text values that appear non-numeric and descriptive (e.g., "Product", "Revenue", "Date") while subsequent rows contain numeric or mixed content, the first row is the header. This is a weak signal and is used only to confirm stronger signals.

### 10.6 Configurable Header Rows

The `headerRows` option allows the caller to explicitly set the number of header rows:

- `headerRows: 0` -- no header row. All rows are data rows. Synthetic headers are generated as `["Column 1", "Column 2", ...]`.
- `headerRows: 1` (default when auto-detection is ambiguous) -- the first row is the header.
- `headerRows: 2` or more -- multiple header rows, flattened into a single header array by concatenating parent and child text.
- `headerRows: 'auto'` (default) -- use the heuristic detection described above.

---

## 11. Merged Cell Detection

Merged cells are cells that span multiple rows or columns. They are common in financial reports (category headers spanning all columns), academic tables (group labels), and forms (address fields spanning multiple columns).

### 11.1 Lattice Mode Merged Cell Detection

In lattice mode, merged cells are detected from missing internal grid lines (Section 6.7).

**Horizontal merge (colspan):** If the vertical grid line between cells `(r, c)` and `(r, c+1)` is absent, the cells are merged. The merge is transitive: if the line between `c+1` and `c+2` is also absent, the span extends to 3 columns.

**Vertical merge (rowspan):** If the horizontal grid line between cells `(r, c)` and `(r+1, c)` is absent, the cells are merged vertically.

**Combined merge:** A cell can span both rows and columns simultaneously. The merge region is the maximal rectangle of cells connected by missing internal grid lines.

### 11.2 Stream Mode Merged Cell Detection

Without grid lines, merged cell detection in stream mode relies on text positioning:

**Horizontal merge detection:**
- A text element whose width spans from the left boundary of column `c` to beyond the right boundary of column `c+1` is a candidate merged cell.
- If the text is centered between the boundaries of columns `c` and `c+1` (its center x-coordinate falls between the boundaries rather than within a single column), it is classified as a merged cell spanning those columns.

**Vertical merge detection:**
- If a cell `(r, c)` contains text and cell `(r+1, c)` is empty but the expected content for row `r+1` is consistent with the value in `(r, c)` being a spanning header, a vertical merge is inferred.
- This is the weakest detection and often requires the `headerRows` option to be explicitly set for reliable results.

### 11.3 Output Representation

In the `cells` output format, merged cells are represented as:

```typescript
{
  text: "Category A",
  row: 2,
  col: 0,
  rowSpan: 3,
  colSpan: 1,
  isHeader: false
}
```

In the flattened `headers`/`rows` output format, the merged cell's text is duplicated into every position it spans. Cell `(2, 0)`, `(3, 0)`, and `(4, 0)` all contain `"Category A"`.

---

## 12. Image-Based Extraction

For scanned PDFs (where pages are images with no extractable text) and standalone image files (JPEG, PNG, TIFF), table extraction requires OCR to first obtain text with positions, then applies stream-mode analysis on the OCR output.

### 12.1 Scanned Page Detection

A PDF page is classified as scanned if:

1. `page.getTextContent()` returns fewer than 10 text elements.
2. The page contains image operators that cover more than 80% of the page area.

When a scanned page is detected, the extraction pipeline switches to image-based extraction for that page.

### 12.2 Page Rendering

The PDF page is rendered to a pixel image at a configurable DPI (default: 300). Higher DPI produces better OCR accuracy but increases memory usage and processing time.

**Implementation:** Use `pdfjs-dist`'s rendering capabilities with a `canvas` implementation (`canvas` npm package for Node.js, which provides a Canvas API outside the browser). The rendered image is a Buffer in PNG format.

For standalone image inputs (JPEG, PNG, TIFF), no rendering step is needed -- the image is used directly.

### 12.3 OCR with tesseract.js

`tesseract.js` is used to recognize text in the rendered image. OCR produces text elements with bounding boxes (x, y, width, height) and confidence scores.

**Configuration:**
- `ocrLanguage`: Language(s) for recognition (default: `'eng'`).
- `ocrDPI`: DPI for page rendering (default: 300).
- `ocrConfidenceThreshold`: Minimum character confidence to include in results (default: 60).

**Output:** The OCR results are converted to the same text element format used by `pdfjs-dist` (text, x, y, width, height, fontSize). This allows the stream-mode algorithm to process OCR output identically to native PDF text.

### 12.4 Stream Mode on OCR Output

After OCR, the positioned text elements are passed to the stream-mode extraction pipeline (Sections 7.2-7.6). The algorithm is identical -- column detection from x-clustering, row detection from y-clustering, cell assignment, header detection.

**Quality considerations:** OCR introduces noise that native PDF extraction does not have:
- Character recognition errors: "0" vs "O", "1" vs "l", "$" vs "S".
- Positional imprecision: OCR bounding boxes are less precise than PDF text positions, leading to noisier column detection.
- Table line interference: Drawn grid lines in scanned images may be recognized as characters or may interfere with text recognition.

The extraction pipeline accounts for these by using wider clustering tolerances for OCR-sourced text elements (1.5x the normal tolerance).

### 12.5 Optional Vision LLM Integration

For cases where heuristic extraction produces unsatisfactory results on complex scanned documents, an optional integration with vision-capable LLMs provides AI-powered table extraction.

**Flow:**
1. The rendered page image is sent to a vision LLM (GPT-4o, Claude Vision, or any model that accepts images and returns structured output).
2. The prompt instructs the LLM to identify all tables in the image and return them as JSON arrays with headers and rows.
3. The LLM response is parsed and converted to `ExtractedTable` objects.

**Configuration:**
- `llmProvider`: The LLM provider configuration (API key, model, endpoint).
- `llmMode`: `'fallback'` (use LLM only when heuristic extraction fails or produces low-confidence results) or `'always'` (always use LLM for image-based pages).

**Trade-offs:** LLM extraction is slower (2-10 seconds per page), costs money (API tokens), requires network access, and is non-deterministic (different runs may produce slightly different results). It is not enabled by default. When enabled in fallback mode, it is invoked only for pages where the heuristic confidence score is below a threshold (default: 0.4).

---

## 13. Output Format

### 13.1 ExtractedTable Object

The primary output of table extraction. Each detected and extracted table produces one `ExtractedTable` object.

```typescript
interface ExtractedTable {
  /** Zero-based index of this table in the extraction results. */
  index: number;

  /** Column header strings. One entry per column. */
  headers: string[];

  /** Data rows. Each row is an array of cell strings with the same length as headers. */
  rows: string[][];

  /** Cell-level representation with metadata. */
  cells: Cell[][];

  /** Metadata about the extraction. */
  metadata: TableMetadata;

  /** Export the table to a specific format. */
  toCSV(): string;
  toMarkdown(): string;
  toHTML(): string;
  toJSON(): { headers: string[]; rows: string[][] };
}

interface Cell {
  /** The text content of the cell. */
  text: string;

  /** Zero-based row index. */
  row: number;

  /** Zero-based column index. */
  col: number;

  /** Number of rows this cell spans. Default: 1. */
  rowSpan: number;

  /** Number of columns this cell spans. Default: 1. */
  colSpan: number;

  /** Whether this cell is a header cell. */
  isHeader: boolean;

  /** Bounding box of this cell in page coordinates (points). */
  bounds?: { x: number; y: number; width: number; height: number };
}

interface TableMetadata {
  /** One-based page number where the table was found. */
  page: number;

  /** Bounding box of the entire table on the page, in points. */
  bounds: { x: number; y: number; width: number; height: number };

  /** Confidence score of the table detection (0.0 to 1.0). */
  confidence: number;

  /** Extraction mode used for this table. */
  mode: 'lattice' | 'stream' | 'ocr' | 'llm';

  /** Number of data rows (excluding header). */
  rowCount: number;

  /** Number of columns. */
  colCount: number;

  /** Whether OCR was applied to extract this table. */
  ocrApplied: boolean;

  /** Whether the table spans multiple pages. */
  multiPage: boolean;

  /** If multiPage, the pages the table spans. */
  pageRange?: [number, number];

  /** Whether merged cells were detected. */
  hasMergedCells: boolean;

  /** Time taken to extract this table, in milliseconds. */
  extractionMs: number;
}
```

### 13.2 CSV Export

`table.toCSV()` produces RFC 4180 compliant CSV:

```
Product,Price,Stock
"Widget A",49.99,240
"Widget B",39.99,85
```

Fields containing commas, double quotes, or newlines are quoted. Double quotes within fields are escaped by doubling (`""`). The header row is always the first line.

### 13.3 Markdown Export

`table.toMarkdown()` produces a GFM pipe table:

```markdown
| Product | Price | Stock |
|---------|-------|-------|
| Widget A | 49.99 | 240 |
| Widget B | 39.99 | 85 |
```

Pipe characters in cell content are escaped (`\|`). The separator row uses `---` for each column. Alignment markers are not added by default (available via `toMarkdown({ alignment: true })`).

### 13.4 HTML Export

`table.toHTML()` produces a complete HTML table:

```html
<table>
  <thead>
    <tr><th>Product</th><th>Price</th><th>Stock</th></tr>
  </thead>
  <tbody>
    <tr><td>Widget A</td><td>49.99</td><td>240</td></tr>
    <tr><td>Widget B</td><td>39.99</td><td>85</td></tr>
  </tbody>
</table>
```

Merged cells include `rowspan` and `colspan` attributes. HTML entities in cell content are escaped.

### 13.5 JSON Export

`table.toJSON()` returns a plain object:

```json
{
  "headers": ["Product", "Price", "Stock"],
  "rows": [
    ["Widget A", "49.99", "240"],
    ["Widget B", "39.99", "85"]
  ]
}
```

This is the minimal, portable representation suitable for database ingestion, API responses, and further processing.

---

## 14. API Surface

### Installation

```bash
npm install doc-table-extract
```

### Main Export: `extractTables`

The primary API. Auto-detects input type (PDF or image), extracts all tables, and returns them.

```typescript
import { extractTables } from 'doc-table-extract';

// From file path
const tables = await extractTables('./report.pdf');
for (const table of tables) {
  console.log(table.headers);
  console.log(table.rows);
}

// From Buffer
const buffer = await fs.promises.readFile('./invoice.pdf');
const tables = await extractTables(buffer);

// From URL
const tables = await extractTables('https://example.com/filing.pdf');

// With options
const tables = await extractTables('./report.pdf', {
  mode: 'lattice',
  pages: [1, 2, 3],
  minConfidence: 0.7,
});
```

**Signature:**

```typescript
function extractTables(
  input: TableInput,
  options?: ExtractOptions,
): Promise<ExtractedTable[]>;
```

### Format-Specific Exports

```typescript
import { extractTablesFromPDF, extractTablesFromImage } from 'doc-table-extract';

// PDF with PDF-specific options
const tables = await extractTablesFromPDF('./10-K.pdf', {
  mode: 'auto',
  pages: [45, 46],
  password: 'secret',
  mergeMultiPageTables: true,
});

// Image with OCR options
const tables = await extractTablesFromImage('./scanned-page.png', {
  ocrLanguage: 'eng',
  ocrDPI: 300,
});
```

### Detection Export: `detectTableRegions`

Detects table regions without extracting cell content. Returns bounding boxes, confidence scores, and detection method for each table.

```typescript
import { detectTableRegions } from 'doc-table-extract';

const regions = await detectTableRegions('./report.pdf');
for (const region of regions) {
  console.log(`Page ${region.page}: table at (${region.bounds.x}, ${region.bounds.y}), confidence: ${region.confidence}`);
}
```

**Signature:**

```typescript
function detectTableRegions(
  input: TableInput,
  options?: DetectOptions,
): Promise<TableRegion[]>;
```

### Factory Export: `createExtractor`

Creates a configured extractor instance with preset options.

```typescript
import { createExtractor } from 'doc-table-extract';

const extractor = createExtractor({
  mode: 'auto',
  minConfidence: 0.6,
  headerRows: 'auto',
  ocr: { enabled: true, language: 'eng' },
});

const tables1 = await extractor.extractTables('./report1.pdf');
const tables2 = await extractor.extractTables('./report2.pdf', { pages: [1] });
const regions = await extractor.detectTableRegions('./report3.pdf');
```

### Type Definitions

```typescript
// ── Input Types ─────────────────────────────────────────────────────

/** Accepted input types for table extraction. */
type TableInput = string | Buffer | URL;

// ── Extract Options ─────────────────────────────────────────────────

/** Options for table extraction. */
interface ExtractOptions {
  /**
   * Extraction mode.
   * 'lattice': Extract tables with visible grid lines only.
   * 'stream': Extract tables without visible grid lines only.
   * 'auto': Detect grid lines and choose the appropriate mode per table.
   * Default: 'auto'.
   */
  mode?: 'lattice' | 'stream' | 'auto';

  /**
   * Specific pages to extract from (1-based).
   * Default: all pages.
   */
  pages?: number[] | { start: number; end: number };

  /**
   * Minimum confidence score to include a table in results.
   * Default: 0.5.
   */
  minConfidence?: number;

  /**
   * Number of header rows, or 'auto' for heuristic detection.
   * Default: 'auto'.
   */
  headerRows?: number | 'auto';

  /**
   * Whether to merge tables that span multiple pages.
   * Default: true.
   */
  mergeMultiPageTables?: boolean;

  /**
   * Whether to preserve newlines in multi-line cells.
   * If false, multi-line cell content is joined with spaces.
   * Default: false.
   */
  preserveCellNewlines?: boolean;

  /**
   * Password for encrypted PDFs.
   */
  password?: string;

  /**
   * OCR options for scanned pages and images.
   */
  ocr?: OCROptions;

  /**
   * Vision LLM options for AI-powered extraction.
   */
  llm?: LLMOptions;

  /**
   * Lattice mode specific options.
   */
  lattice?: LatticeOptions;

  /**
   * Stream mode specific options.
   */
  stream?: StreamOptions;

  /**
   * AbortSignal for external cancellation.
   */
  signal?: AbortSignal;
}

/** Options specific to lattice mode. */
interface LatticeOptions {
  /**
   * Tolerance in points for classifying a line as horizontal or vertical.
   * Default: 2.
   */
  lineTolerance?: number;

  /**
   * Minimum line length in points to be considered a grid line.
   * Default: 20.
   */
  minLineLength?: number;

  /**
   * Snap tolerance for aligning nearby lines to the same coordinate.
   * Default: 3.
   */
  snapTolerance?: number;

  /**
   * Maximum line width in points. Lines thicker than this are
   * treated as borders/decorations, not grid lines.
   * Default: 5.
   */
  maxLineWidth?: number;
}

/** Options specific to stream mode. */
interface StreamOptions {
  /**
   * Minimum horizontal gap (in multiples of average character width)
   * to consider as a column separator.
   * Default: 3.
   */
  columnGapMultiplier?: number;

  /**
   * Minimum vertical gap (in multiples of line height)
   * to consider as a row separator.
   * Default: 0.6.
   */
  rowGapMultiplier?: number;

  /**
   * Minimum number of consecutive aligned rows to detect a table.
   * Default: 3.
   */
  minAlignedRows?: number;

  /**
   * Minimum number of columns to detect a table.
   * Default: 2.
   */
  minColumns?: number;
}

/** OCR options for scanned PDFs and images. */
interface OCROptions {
  /**
   * Whether to enable OCR for scanned pages.
   * Requires tesseract.js as a peer dependency.
   * Default: false.
   */
  enabled?: boolean;

  /**
   * Language(s) for OCR recognition.
   * Default: 'eng'.
   */
  language?: string;

  /**
   * DPI for rendering PDF pages before OCR.
   * Default: 300.
   */
  dpi?: number;

  /**
   * Minimum character confidence (0-100) to include in results.
   * Default: 60.
   */
  confidenceThreshold?: number;
}

/** Vision LLM options for AI-powered extraction. */
interface LLMOptions {
  /**
   * When to use the LLM.
   * 'fallback': Only when heuristic extraction produces low-confidence results.
   * 'always': Always use LLM for image-based pages.
   * Default: 'fallback'.
   */
  mode?: 'fallback' | 'always';

  /**
   * Confidence threshold below which the LLM is invoked in fallback mode.
   * Default: 0.4.
   */
  fallbackThreshold?: number;

  /**
   * Function that sends an image to the LLM and returns extracted tables.
   * The caller provides this function, allowing any LLM provider.
   */
  extract: (imageBuffer: Buffer, prompt: string) => Promise<{ headers: string[]; rows: string[][] }[]>;
}

// ── Detection Types ─────────────────────────────────────────────────

/** Options for table region detection. */
interface DetectOptions {
  /**
   * Detection mode.
   * Default: 'auto'.
   */
  mode?: 'lattice' | 'stream' | 'auto';

  /**
   * Pages to scan.
   * Default: all pages.
   */
  pages?: number[] | { start: number; end: number };

  /**
   * Minimum confidence to include a region.
   * Default: 0.3 (lower than extraction default, since detection is exploratory).
   */
  minConfidence?: number;

  /** Password for encrypted PDFs. */
  password?: string;
}

/** A detected table region on a page. */
interface TableRegion {
  /** One-based page number. */
  page: number;

  /** Bounding box in page coordinates (points). */
  bounds: { x: number; y: number; width: number; height: number };

  /** Detection confidence (0.0 to 1.0). */
  confidence: number;

  /** How the region was detected. */
  method: 'lattice' | 'text-cluster';

  /** Estimated row count (from detection, may differ from extraction). */
  estimatedRows: number;

  /** Estimated column count. */
  estimatedColumns: number;
}

// ── Extractor Instance ──────────────────────────────────────────────

/** A configured extractor instance created by createExtractor(). */
interface TableExtractor {
  /** Extract tables with preset options. */
  extractTables(input: TableInput, overrides?: Partial<ExtractOptions>): Promise<ExtractedTable[]>;

  /** Extract tables from PDF with preset options. */
  extractTablesFromPDF(input: TableInput, overrides?: Partial<ExtractOptions>): Promise<ExtractedTable[]>;

  /** Extract tables from image with preset options. */
  extractTablesFromImage(input: TableInput, overrides?: Partial<ExtractOptions>): Promise<ExtractedTable[]>;

  /** Detect table regions with preset options. */
  detectTableRegions(input: TableInput, overrides?: Partial<DetectOptions>): Promise<TableRegion[]>;
}
```

### Example: Financial Report Table Extraction

```typescript
import { extractTables } from 'doc-table-extract';

const tables = await extractTables('./annual-report.pdf', {
  mode: 'auto',
  pages: { start: 40, end: 60 },
  minConfidence: 0.7,
  headerRows: 'auto',
  mergeMultiPageTables: true,
});

for (const table of tables) {
  console.log(`Table on page ${table.metadata.page}:`);
  console.log(`  Mode: ${table.metadata.mode}`);
  console.log(`  Confidence: ${table.metadata.confidence}`);
  console.log(`  Headers: ${table.headers.join(' | ')}`);
  console.log(`  Rows: ${table.metadata.rowCount}`);
  console.log(`  CSV:\n${table.toCSV()}`);
}
```

### Example: Scanned Invoice with OCR

```typescript
import { extractTables } from 'doc-table-extract';

const tables = await extractTables('./scanned-invoice.pdf', {
  ocr: {
    enabled: true,
    language: 'eng',
    dpi: 300,
  },
});

const lineItems = tables[0];
console.log(lineItems.headers);
// ['Item', 'Quantity', 'Unit Price', 'Total']
console.log(lineItems.rows);
// [['Widget A', '10', '$49.99', '$499.90'], ...]
```

### Example: Vision LLM Fallback

```typescript
import { extractTables } from 'doc-table-extract';
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

const tables = await extractTables('./complex-report.pdf', {
  ocr: { enabled: true },
  llm: {
    mode: 'fallback',
    fallbackThreshold: 0.4,
    extract: async (imageBuffer, prompt) => {
      const response = await anthropic.messages.create({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 4096,
        messages: [{
          role: 'user',
          content: [
            { type: 'image', source: { type: 'base64', media_type: 'image/png', data: imageBuffer.toString('base64') } },
            { type: 'text', text: prompt },
          ],
        }],
      });
      return JSON.parse(response.content[0].text);
    },
  },
});
```

---

## 15. Configuration

### Default Values

| Option | Default | Description |
|--------|---------|-------------|
| `mode` | `'auto'` | Auto-detect lattice or stream per table |
| `pages` | All pages | Pages to extract from |
| `minConfidence` | `0.5` | Minimum confidence to include a table |
| `headerRows` | `'auto'` | Heuristic header detection |
| `mergeMultiPageTables` | `true` | Merge tables spanning page boundaries |
| `preserveCellNewlines` | `false` | Join multi-line cell content with spaces |
| `ocr.enabled` | `false` | OCR disabled by default |
| `ocr.language` | `'eng'` | English OCR |
| `ocr.dpi` | `300` | 300 DPI for page rendering |
| `ocr.confidenceThreshold` | `60` | Minimum OCR character confidence |
| `lattice.lineTolerance` | `2` | 2 points tolerance for horizontal/vertical classification |
| `lattice.minLineLength` | `20` | 20 points minimum line length |
| `lattice.snapTolerance` | `3` | 3 points snap tolerance |
| `lattice.maxLineWidth` | `5` | 5 points maximum grid line width |
| `stream.columnGapMultiplier` | `3` | 3x character width for column gap |
| `stream.rowGapMultiplier` | `0.6` | 0.6x line height for row gap |
| `stream.minAlignedRows` | `3` | 3 minimum consecutive aligned rows |
| `stream.minColumns` | `2` | 2 minimum columns to detect a table |

### Multi-Page Table Merging

When `mergeMultiPageTables` is enabled, the extractor detects tables that continue across page boundaries:

1. A table at the bottom of page N and a table at the top of page N+1 are merge candidates if:
   - They have the same number of columns.
   - Column boundary x-coordinates match within tolerance.
   - The top table does not end with a clear termination signal (a thick bottom border, a "Total" row).
   - The bottom table does not start with a new header row (unless the document repeats headers on each page, detected by matching header content).
2. If headers are repeated on page N+1, the duplicate header row is removed.
3. The merged table's `metadata.multiPage` is `true`, and `metadata.pageRange` is `[N, N+1]`.

---

## 16. CLI

### Installation and Invocation

```bash
# Global install
npm install -g doc-table-extract
doc-table-extract report.pdf

# npx (no install)
npx doc-table-extract invoice.pdf --format csv

# Package script
# package.json: { "scripts": { "extract": "doc-table-extract" } }
npm run extract -- report.pdf
```

### CLI Binary Name

`doc-table-extract`

### Commands and Flags

```
doc-table-extract <input> [options]

Positional:
  <input>                    File path or URL to a PDF or image.

Extraction options:
  --mode <mode>              Extraction mode. Values: auto, lattice, stream. Default: auto.
  --pages <pages>            Pages to extract (comma-separated or range). Example: 1,2,5 or 1-10.
  --min-confidence <n>       Minimum confidence threshold (0.0-1.0). Default: 0.5.
  --header-rows <n>          Number of header rows, or 'auto'. Default: auto.
  --no-merge                 Do not merge tables spanning multiple pages.
  --password <pwd>           Password for encrypted PDFs.

OCR options:
  --ocr                      Enable OCR for scanned pages.
  --ocr-language <lang>      OCR language. Default: eng.
  --ocr-dpi <dpi>            OCR rendering DPI. Default: 300.

Output options:
  --format <format>          Output format. Values: json, csv, markdown, html. Default: json.
  --output <path>            Write output to file instead of stdout.
  --pretty                   Pretty-print JSON output.
  --table <n>                Extract only the Nth table (0-based). Default: all tables.
  --detect-only              Only detect table regions, do not extract content.

Meta:
  --version                  Print version and exit.
  --help                     Print help and exit.
```

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success. Tables extracted and written to output. |
| `1` | No tables found. The input was processed but no tables were detected. |
| `2` | Input error. File not found, unsupported format, or invalid options. |
| `3` | Extraction error. A fatal error occurred during extraction. |

### Output Examples

**JSON output (default):**

```bash
$ doc-table-extract report.pdf --pretty
```

```json
[
  {
    "index": 0,
    "headers": ["Product", "Price", "Stock"],
    "rows": [
      ["Widget A", "$49.99", "240"],
      ["Widget B", "$39.99", "85"]
    ],
    "metadata": {
      "page": 3,
      "confidence": 0.92,
      "mode": "lattice",
      "rowCount": 2,
      "colCount": 3
    }
  }
]
```

**CSV output:**

```bash
$ doc-table-extract invoice.pdf --format csv --table 0
```

```
Product,Price,Stock
Widget A,$49.99,240
Widget B,$39.99,85
```

**Markdown output:**

```bash
$ doc-table-extract report.pdf --format markdown
```

```markdown
| Product | Price | Stock |
|---------|-------|-------|
| Widget A | $49.99 | 240 |
| Widget B | $39.99 | 85 |
```

**Detection-only output:**

```bash
$ doc-table-extract report.pdf --detect-only --pretty
```

```json
[
  {
    "page": 3,
    "bounds": { "x": 72, "y": 200, "width": 468, "height": 150 },
    "confidence": 0.92,
    "method": "lattice",
    "estimatedRows": 5,
    "estimatedColumns": 3
  }
]
```

---

## 17. Integration

### With docling-node-ts

`docling-node-ts` converts full documents to markdown and includes basic table detection. `doc-table-extract` provides a specialized, higher-accuracy table extraction engine. The integration:

```typescript
import { convert } from 'docling-node-ts';
import { extractTables } from 'doc-table-extract';

// Use docling-node-ts for the full document
const result = await convert('./report.pdf');
console.log(result.markdown);

// Use doc-table-extract for precise table data
const tables = await extractTables('./report.pdf', {
  pages: result.tables.map(t => t.page),
});
// tables provides cell-level metadata, merged cell detection,
// and higher accuracy than docling-node-ts's built-in table detection
```

### With table-chunk

`table-chunk` chunks already-extracted tables for RAG. `doc-table-extract` extracts the tables in the first place. The pipeline:

```typescript
import { extractTables } from 'doc-table-extract';
import { chunkTable } from 'table-chunk';

const tables = await extractTables('./report.pdf');
for (const table of tables) {
  const chunks = chunkTable({
    headers: table.headers,
    rows: table.rows,
  }, {
    strategy: 'row',
    rowsPerChunk: 10,
    outputFormat: 'markdown',
  });
  // Each chunk is a markdown table with headers, ready for embedding
}
```

### With ai-file-router

`ai-file-router` detects file types and routes them to the appropriate processor. PDF files containing tables can be routed to `doc-table-extract`:

```typescript
import { routeFile } from 'ai-file-router';
import { extractTables } from 'doc-table-extract';

const route = await routeFile('./uploaded-file.pdf');
if (route.type === 'pdf') {
  const tables = await extractTables(route.path);
  // Process extracted tables
}
```

### With rag-prompt-builder

`rag-prompt-builder` composes retrieved context into LLM prompts. Extracted tables can be formatted as context:

```typescript
import { extractTables } from 'doc-table-extract';
import { buildPrompt } from 'rag-prompt-builder';

const tables = await extractTables('./10-K.pdf');
const tableContext = tables.map(t => t.toMarkdown()).join('\n\n');

const prompt = buildPrompt({
  query: 'What was the revenue in Q3?',
  context: [tableContext],
});
```

---

## 18. Testing Strategy

### Unit Tests

Unit tests isolate individual algorithms and modules.

- **Line detection tests**: Provide mock operator lists with known line segments. Verify that horizontal and vertical lines are correctly extracted, classified, and snapped.
- **Intersection detection tests**: Provide sets of horizontal and vertical lines with known intersections. Verify that all intersections are found and deduplicated.
- **Grid construction tests**: Provide intersections that form known grids. Verify that the correct number of rows, columns, and cells are produced.
- **Column detection tests (stream mode)**: Provide mock text elements with known x-coordinates. Verify that columns are correctly detected from x-coordinate clustering for left-aligned, right-aligned, and center-aligned layouts.
- **Row detection tests (stream mode)**: Provide mock text elements with known y-coordinates. Verify row clustering and multi-line row merging.
- **Cell assignment tests**: Provide a known grid and mock text elements. Verify that each element is assigned to the correct cell.
- **Header detection tests**: Provide tables with bold first rows, larger first rows, underlined first rows, and ambiguous cases. Verify that headers are correctly identified.
- **Merged cell detection tests**: Provide grids with missing internal lines (lattice) and spanning text elements (stream). Verify correct colspan and rowspan values.
- **Confidence scoring tests**: Provide tables with varying quality signals. Verify that scores match expected ranges.
- **CSV/markdown/HTML export tests**: Provide known `ExtractedTable` objects. Verify that exported strings match expected output exactly.

### Integration Tests with Real PDFs

Integration tests use real PDF fixtures to verify end-to-end extraction accuracy.

- **Lattice table fixture**: A PDF containing a ruled table (visible grid lines). Verify that all cells are correctly extracted, headers are identified, and the cell content matches expected values.
- **Stream table fixture**: A PDF containing a borderless table (no grid lines). Verify column detection accuracy and cell assignment.
- **Multi-table fixture**: A PDF page containing two tables. Verify that both are detected as separate tables.
- **Multi-page table fixture**: A table spanning two pages. Verify that merge produces a single table with the correct total row count.
- **Merged cell fixture**: A PDF with tables containing colspan and rowspan cells. Verify that spans are correctly detected and the flattened output is correct.
- **Mixed mode fixture**: A PDF with some pages having ruled tables and other pages having borderless tables. Verify that auto mode selects the correct mode per page.

### Benchmark Against Camelot/Tabula

For each PDF fixture, extract tables using Camelot (Python) and Tabula (Java) for comparison. The test suite verifies that `doc-table-extract` produces equivalent results for the standard test cases. Differences are documented and analyzed. The goal is not identical output (different tools make different formatting choices) but equivalent data extraction accuracy: the same cells with the same values in the same positions.

### Test Framework

Tests use Vitest, matching the project's configuration in `package.json`. PDF fixtures are stored in `test/fixtures/` as real PDF files committed to the repository.

---

## 19. Performance

### Extraction Speed

Target performance benchmarks for a standard developer machine (M-series Mac, 16GB RAM):

| Scenario | Target | Notes |
|----------|--------|-------|
| Single-page PDF, 1 lattice table | < 200ms | Grid line detection + text extraction |
| Single-page PDF, 1 stream table | < 300ms | Text clustering is more compute-intensive |
| 10-page PDF, 3 tables | < 1 second | Sequential page processing |
| 100-page PDF, 15 tables | < 5 seconds | Most time spent in pdfjs-dist text extraction |
| Scanned page with OCR | 2-10 seconds per page | Dominated by tesseract.js OCR time |
| Vision LLM extraction | 2-15 seconds per page | Dominated by API round-trip latency |

### Memory Usage

- **PDF parsing**: `pdfjs-dist` loads pages on demand. Only one page's text content is in memory at a time during sequential processing. Peak memory is proportional to the largest page's text content.
- **OCR**: Page rendering at 300 DPI for a standard letter-size page produces a ~25MB raw image buffer. This is the peak memory for OCR processing. Only one page is rendered at a time.
- **Large tables**: A table with 1000 rows and 10 columns produces approximately 10,000 string cells. At an average of 20 characters per cell, this is ~200KB -- negligible.
- **Multi-page PDFs**: A 500-page PDF is processed page by page. Memory does not scale with total page count, only with per-page complexity. The results array accumulates `ExtractedTable` objects, which are lightweight (headers + rows + metadata).

### Streaming

For very large documents, the extractor processes pages sequentially and yields results as they are found. The `extractTables` function processes all pages and returns the complete results array. A future streaming API could yield tables as they are extracted from each page, but this is not in scope for v1.

---

## 20. Dependencies

### Runtime Dependencies

| Dependency | Purpose | Why Required |
|-----------|---------|-------------|
| `pdfjs-dist` | PDF parsing and text extraction with positions. Provides `getDocument()`, `page.getTextContent()`, and `page.getOperatorList()` for accessing text elements, font metadata, and vector graphics. | The only maintained, production-quality JavaScript PDF parser. No alternative provides text positioning and operator list access. `pdf-parse` wraps `pdfjs-dist` but strips position data. |

### Peer Dependencies (Optional)

| Dependency | Purpose | When Required |
|-----------|---------|--------------|
| `tesseract.js` | OCR for scanned PDFs and images. | Only when `ocr.enabled` is `true`. Not needed for digital PDFs with extractable text. |
| `canvas` | Node.js Canvas API for rendering PDF pages to images (needed for OCR). | Only when OCR is enabled. `pdfjs-dist` requires a Canvas implementation for rendering. |

### Dev Dependencies

| Dependency | Purpose |
|-----------|---------|
| `typescript` | TypeScript compiler |
| `vitest` | Test runner |
| `eslint` | Linter |

### No CLI Framework

CLI argument parsing uses Node.js built-in `util.parseArgs` (available since Node.js 18). No external CLI framework (commander, yargs, meow) is required.

---

## 21. File Structure

```
doc-table-extract/
  package.json
  tsconfig.json
  SPEC.md
  README.md
  src/
    index.ts                    # Public API exports
    extractor.ts                # Main extraction orchestrator
    types.ts                    # All TypeScript type definitions
    detect/
      detector.ts               # Table region detection coordinator
      line-detector.ts          # Ruled line extraction from PDF operator stream
      text-cluster-detector.ts  # Text cluster-based table detection
      confidence.ts             # Confidence score computation
    extract/
      lattice.ts                # Lattice mode extraction pipeline
      stream.ts                 # Stream mode extraction pipeline
      grid.ts                   # Grid construction from intersections
      columns.ts                # Column detection algorithm
      rows.ts                   # Row detection algorithm
      headers.ts                # Header row detection
      merged-cells.ts           # Merged cell detection
      multi-page.ts             # Multi-page table merging
    pdf/
      pdf-parser.ts             # pdfjs-dist wrapper for text and operator extraction
      text-elements.ts          # Text element normalization and positioning
      font-analyzer.ts          # Font metadata analysis (bold, size, family)
      line-extractor.ts         # Line segment extraction from operator list
    image/
      image-extractor.ts        # Image-based extraction pipeline
      ocr.ts                    # tesseract.js OCR integration
      llm.ts                    # Vision LLM integration
      renderer.ts               # PDF page-to-image rendering
    output/
      csv.ts                    # CSV export
      markdown.ts               # Markdown export
      html.ts                   # HTML export
      json.ts                   # JSON export
    cli.ts                      # CLI entry point
  test/
    fixtures/                   # Real PDF test fixtures
      lattice-table.pdf
      stream-table.pdf
      multi-table.pdf
      multi-page-table.pdf
      merged-cells.pdf
      scanned-page.pdf
    unit/
      line-detector.test.ts
      text-cluster-detector.test.ts
      columns.test.ts
      rows.test.ts
      headers.test.ts
      merged-cells.test.ts
      confidence.test.ts
      lattice.test.ts
      stream.test.ts
      csv-export.test.ts
      markdown-export.test.ts
      html-export.test.ts
    integration/
      extract-lattice.test.ts
      extract-stream.test.ts
      extract-auto.test.ts
      multi-page.test.ts
      ocr.test.ts
      cli.test.ts
  dist/                         # Compiled output (gitignored)
```

---

## 22. Implementation Roadmap

### Phase 1: Core PDF Text Extraction (v0.1.0)

- Implement `pdfjs-dist` wrapper for text element extraction with positions.
- Implement font metadata analysis (bold detection, size extraction).
- Implement basic text element normalization (coordinate normalization, reading order).
- Implement the `ExtractedTable` type and JSON export.
- Write unit tests for text extraction and font analysis.

### Phase 2: Lattice Mode (v0.2.0)

- Implement line segment extraction from PDF operator lists.
- Implement horizontal/vertical line classification and snapping.
- Implement intersection detection and grid construction.
- Implement text-to-cell assignment.
- Implement header detection (font weight, font size, ruled line signals).
- Implement merged cell detection from missing grid lines.
- Implement CSV and markdown export.
- Write unit and integration tests with lattice table fixtures.

### Phase 3: Stream Mode (v0.3.0)

- Implement column detection from x-coordinate clustering.
- Implement row detection from y-coordinate clustering.
- Implement multi-line cell handling.
- Implement text cluster-based table detection.
- Implement confidence scoring.
- Write unit and integration tests with stream table fixtures.

### Phase 4: Auto Mode and Multi-Table (v0.4.0)

- Implement auto mode (select lattice or stream per page/table).
- Implement multi-table detection on a single page.
- Implement multi-page table merging.
- Implement table region detection API (`detectTableRegions`).
- Implement the factory API (`createExtractor`).

### Phase 5: Image and OCR Support (v0.5.0)

- Implement scanned page detection.
- Implement PDF page rendering to image.
- Implement tesseract.js OCR integration.
- Implement stream mode on OCR output with adjusted tolerances.
- Implement standalone image input (JPEG, PNG, TIFF).
- Write integration tests with scanned fixtures.

### Phase 6: CLI and Polish (v0.6.0)

- Implement CLI with all flags and output formats.
- Implement HTML export.
- Implement vision LLM integration (optional).
- Performance optimization and benchmarking.
- Comprehensive integration testing.
- Write README.

### Phase 7: Production Release (v1.0.0)

- Benchmark against Camelot and Tabula on standard test PDFs.
- Fix accuracy gaps identified in benchmarking.
- API stabilization and documentation review.
- Publish to npm.

---

## 23. Example Use Cases

### Financial Report Tables

Extract income statements, balance sheets, and cash flow statements from SEC 10-K filings. These are typically lattice tables with clear grid lines, consistent numeric formatting, and multi-level headers ("Fiscal Year Ended December 31" spanning "2024" and "2023" sub-columns).

```typescript
const tables = await extractTables('./AAPL-10K.pdf', {
  mode: 'lattice',
  pages: { start: 40, end: 55 },
  headerRows: 2,
});

const incomeStatement = tables.find(t =>
  t.headers.some(h => h.toLowerCase().includes('revenue'))
);
console.log(incomeStatement?.toCSV());
```

### Invoice Line Item Extraction

Extract line items from PDF invoices for automated accounts payable processing. Invoices have ruled tables with item descriptions, quantities, unit prices, and totals.

```typescript
const tables = await extractTables('./invoice-2025-001.pdf');
const lineItems = tables[0];

for (const row of lineItems.rows) {
  const [description, qty, unitPrice, total] = row;
  await db.insertLineItem({ description, qty: parseInt(qty), unitPrice, total });
}
```

### Academic Paper Result Tables

Extract experimental results from research papers. Papers typically use borderless tables with stream-mode alignment. Results are often compared across methods with numeric metrics.

```typescript
const tables = await extractTables('./paper.pdf', {
  mode: 'stream',
  minConfidence: 0.6,
});

for (const table of tables) {
  console.log(`Table: ${table.headers.join(' | ')}`);
  console.log(`Rows: ${table.metadata.rowCount}`);
}
```

### Scanned Document Digitization

Extract tables from scanned legacy documents for digital archival and data migration.

```typescript
const tables = await extractTables('./scanned-report.pdf', {
  ocr: { enabled: true, language: 'eng', dpi: 300 },
  minConfidence: 0.4,
});

for (const table of tables) {
  const json = table.toJSON();
  await fs.promises.writeFile(
    `./output/table-p${table.metadata.page}.json`,
    JSON.stringify(json, null, 2)
  );
}
```

### Batch Processing Pipeline

Process a directory of PDFs and extract all tables to a structured output directory.

```typescript
import { createExtractor } from 'doc-table-extract';
import { chunkTable } from 'table-chunk';
import { embed } from 'embed-cache';

const extractor = createExtractor({
  mode: 'auto',
  minConfidence: 0.6,
  ocr: { enabled: true },
});

for (const file of pdfFiles) {
  const tables = await extractor.extractTables(file);
  for (const table of tables) {
    const chunks = chunkTable(table.toJSON(), { strategy: 'row', rowsPerChunk: 5 });
    const embeddings = await embed(chunks.map(c => c.text));
    await vectorStore.upsert(embeddings);
  }
}
```

### Form Processing

Extract filled-in data from PDF forms where form fields are rendered as table cells. Government forms, insurance claims, and application forms often use grid-based layouts that lattice mode handles directly.

```typescript
const tables = await extractTables('./tax-form-1099.pdf', {
  mode: 'lattice',
  headerRows: 0,
});

// Form fields are extracted as table cells
// Row/column positions correspond to form field positions
for (const table of tables) {
  for (const row of table.cells) {
    for (const cell of row) {
      if (cell.text.trim()) {
        console.log(`Field at (${cell.row}, ${cell.col}): ${cell.text}`);
      }
    }
  }
}
```
