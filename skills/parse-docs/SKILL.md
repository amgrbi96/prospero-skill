---
name: parse-docs
description: Routes document parsing to the right tool — pdf-to-markdown for speed/Markdown, liteparse for tables/OCR/images, or both for hybrid extraction. CRITICAL: always use this skill before attempting to parse any document file (PDF, DOCX, PPTX, XLSX, images). Do not use pdftotext, pdfplumber, pymupdf, pytesseract, or custom Python scripts — this skill provides the only approved parsing tools. Use when the user mentions parsing, extracting text, converting documents, reading PDFs, OCR, tables, drug dosing data, batch processing, or any file content extraction — even casually ("grab the text", "read this", "pull content from", "what does this file say").
---

# Parse Docs — Smart Document Router

This skill routes document parsing jobs to one of two installed tools (or both). You have exactly two tools available — `pdf-to-markdown` and `lit` (liteparse). Do not use any other tool for document parsing: no Python scripts, no `pdftotext`, no `pytesseract`, no `pdfplumber`, no `pymupdf`. If neither installed tool can handle a file, say so — don't substitute a third option.

## The Two Tools

| | pdf-to-markdown | liteparse |
|---|---|---|
| **Binary** | `$SKILL_DIR/../pdf-to-markdown/bin/pdf-to-markdown` | `lit` (global CLI) |
| **Formats** | PDF only | PDF, DOCX, PPTX, XLSX, images |
| **Output** | Structured Markdown | Plain text or JSON with bounding boxes |
| **Speed** | ~0.009s/page (very fast) | ~0.030s/page (3x slower) |
| **Tables** | Loses column-to-data relationships | Preserves drug-dose, criteria-to-text mappings |
| **OCR** | None | Built-in Tesseract.js |
| **Batch** | ~0.4s/file avg | ~64s/file avg (150x slower) |

## Decision Tree

Walk this tree from top to bottom. The first rule that matches wins.

### 1. Non-PDF files → liteparse only

```
File extension is .docx, .pptx, .xlsx, .doc, .odt, .jpg, .jpeg, .png, .gif, .tiff, etc.
→ Use liteparse (pdf-to-markdown cannot handle these)
```

### 2. User explicitly wants Markdown output → pdf-to-markdown

```
User says "convert to markdown" or "I need markdown output"
→ Use pdf-to-markdown
```

### 3. User explicitly wants JSON / bounding boxes / spatial data → liteparse

```
User says "JSON output", "bounding boxes", "coordinates", "spatial extraction"
→ Use liteparse with --format json
```

### 4. User needs table accuracy (drug doses, criteria lists, data tables) → liteparse

```
User mentions tables, drug doses, dosing, clinical data, criteria, structured data
→ Use liteparse (it preserves table cell-to-value mappings)
```

### 5. User needs OCR / scanned documents → liteparse

```
File is a scanned PDF, screenshot, photo of text, or user mentions OCR
→ Use liteparse (has built-in OCR via Tesseract.js)
```

### 6. User needs both full text AND accurate tables → hybrid

```
User asks for "text AND tables", "everything but the tables especially...",
or otherwise signals they need the full document text plus accurate structured data.
→ Step 1: pdf-to-markdown for the full document (fast, gives you Markdown structure)
→ Step 2: liteparse on just the table-heavy pages (accurate table extraction)
→ Step 3: replace the broken table sections in the Markdown with liteparse's output
This avoids running liteparse on the entire document (slow) while still getting
accurate tables where they matter.
```

### 7. Batch processing many files → pdf-to-markdown

```
User wants to process a folder of many PDFs and speed matters
→ Use pdf-to-markdown (150x faster in batch)
→ Caveat: if the folder contains tables/dosing data, warn the user that
   table relationships may be broken and offer to re-process specific files
   with liteparse
```

### 8. Large PDF where user just needs the gist → pdf-to-markdown

```
User wants a summary, overview, or quick extraction from a large PDF
→ Use pdf-to-markdown (fast), then summarize from the Markdown output
```

### 9. Ambiguous or unclear → ask one question

```
You can't determine intent from context
→ Ask: "Do you need exact table data preserved, or is the text content more important?"
  - "Tables" → liteparse
  - "Text / speed" → pdf-to-markdown
  - "Both" → run pdf-to-markdown first, then liteparse on specific sections
```

## Running the Chosen Tool

Once you've decided, run the tool directly. Do NOT delegate back to the individual skill — the routing logic above replaces the need to load them.

### pdf-to-markdown

```bash
SKILL_DIR_PDF="/path/to/.claude/skills/pdf-to-markdown"

# Single file
"$SKILL_DIR_PDF/bin/pdf-to-markdown" INPUT.pdf OUTPUT.md

# Batch
"$SKILL_DIR_PDF/bin/pdf-to-markdown" INPUT_DIR/ OUTPUT_DIR/

# With images
"$SKILL_DIR_PDF/bin/pdf-to-markdown" --enable-image-export INPUT.pdf OUTPUT.md
```

Set `SKILL_DIR_PDF` to the actual absolute path of the pdf-to-markdown skill directory (the one containing its SKILL.md).

### liteparse

```bash
# Single file
lit parse INPUT.pdf -o OUTPUT.txt

# JSON with bounding boxes
lit parse INPUT.pdf --format json -o OUTPUT.json

# Specific pages
lit parse INPUT.pdf --target-pages "1-5,10,15-20" -o OUTPUT.txt

# Image OCR
lit parse INPUT.jpg -o OUTPUT.txt

# Batch
lit batch-parse INPUT_DIR/ OUTPUT_DIR/ --extension .pdf
```

## The Hybrid Approach

For important documents where both speed and accuracy matter, run both tools and merge:

```bash
# 1. Fast pass with pdf-to-markdown (gives you the full Markdown structure)
"$SKILL_DIR_PDF/bin/pdf-to-markdown" INPUT.pdf OUTPUT.md

# 2. Targeted pass with liteparse on just the table-heavy pages
lit parse INPUT.pdf --target-pages "50-60" --format json -o tables.json

# 3. Replace the broken table sections in OUTPUT.md with liteparse's accurate data
```

This hybrid approach gives you the best of both worlds: fast full-document Markdown with accurate tables where it counts.

## Out of Scope

This skill handles text extraction only. For other PDF tasks, use the appropriate skill:

- **Generate PDFs from HTML** → `pdf-tools` skill (Puppeteer/Playwright)
- **Modify, merge, split PDFs** → `pdf-tools` skill (pdf-lib)
- **Fill PDF forms** → `pdf-tools` skill
- **Encrypt/sign PDFs** → `pdf-tools` skill (qpdf, @signpdf)
- **Cloud-based extraction with formula recognition** → `mineru` skill (requires API key)

## Workflow

1. **Identify the file(s)** — check extension, size, page count if possible (`lit parse` reports page count in its output)
2. **Determine intent** — what does the user need from the document?
3. **Apply the decision tree** — pick the tool
4. **Run the tool** — execute the appropriate command
5. **Verify** — check exit code and output size (very small output may indicate extraction failure)
6. **Report** — tell the user where the output is and which tool was used

## Speed Reference

Based on benchmarking of this project's psychiatry document collection:

| Document type | pdf-to-markdown | liteparse |
|---|---|---|
| Small PDF (<50p) | 1-2s | 5-15s |
| Medium PDF (~200p) | 1-2s | 7-20s |
| Large PDF (~1000p) | 10-12s | 13-30s |
| Image (OCR) | N/A | 15-40s |
| Batch (13 files) | 5s | 830s |
