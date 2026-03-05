---
name: landingai-ade-api
description: "REST API specification for LandingAI's Agentic Document Extraction (ADE). Covers all endpoints (Parse, Extract, Split, Parse Jobs), request parameters, response structures, data types, error codes, model versions, and curl examples."
languages: http/bash
versions: v1
updated-on: 2026-03-04
source: maintainer
tags: landingai,ade,api,document-extraction,parse,extract,split,parse-jobs,curl,rest
---

# LandingAI ADE API Specification

Complete API specification for LandingAI's Agentic Document Extraction (ADE).

## Overview

ADE provides a REST API for document parsing, splitting, data extraction, and large file parse jobs. All SDKs and tools (Python, TypeScript) use this same underlying API.

## Base Configuration

| Region | Base URL |
|--------|----------|
| US (default) | `https://api.va.landing.ai/v1/ade` |
| EU | `https://api.va.eu-west-1.landing.ai/v1/ade` |

**Authentication**: All requests require `Authorization: Bearer $VISION_AGENT_API_KEY`

## SDK Quick Start

```bash
# Python
pip install landingai-ade

# TypeScript / JavaScript
npm install landingai-ade
```

## API Endpoints

### 1. Parse API

**Endpoint**: `POST /parse`

Converts documents to structured json with visual grounding.

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `document` | file | One required | Local file — PDF, images (JPG/PNG/TIFF/WEBP/GIF/BMP/PSD + more), Word (DOC/DOCX/ODT), PowerPoint (PPT/PPTX/ODP), spreadsheets (XLSX/CSV) |
| `document_url` | string | One required | Remote document URL |
| `model` | string | No | Model version (default: `dpt-2-latest`) |
| `split` | string | No | Split mode: `"page"` to split by pages |

#### Response Structure

```json
{
  "markdown": "string",          // Complete document as markdown
  "chunks": [                    // Array of content blocks
    {
      "id": "uuid",              // Unique identifier
      "type": "text, table, marginalia, figure, scan_code, logo, card and attestation", //Chunk type
      "markdown": "string",      // Chunk content
      "grounding": {             // Location in document
        "page": 0,               // Zero-indexed page number
        "box": {                 // Normalized coordinates (0-1)
          "left": 0.1,
          "top": 0.2,
          "right": 0.9,
          "bottom": 0.3
        }
      }
    }
  ],
  "grounding": {                 // Detailed location mapping
    "chunk-id": {              // Chunk grounding (has "chunk" prefix)
      "type": "chunkText|chunkTable|chunkFigure|chunkLogo|chunkCard|chunkAttestation| chunkScanCode|chunkForm|chunkMarginalia| chunkTitle|chunkPageHeader|chunkPageFooter| chunkPageNumber|chunkKeyValue|table|tableCell",
      "page": 0,
      "box": { /* BoundingBox */ }
    },
    "0-1": {                     // Table grounding (format: page-id)
      "type": "table",
      "page": 0,
      "box": { /* BoundingBox */ }
    },
    "0-2": {                     // Table cell grounding
      "type": "tableCell",
      "page": 0,
      "box": { /* BoundingBox */ },
      "position": {              // Cell position data
        "row": 0,
        "col": 0,
        "rowspan": 1,
        "colspan": 1,
        "chunk_id": "uuid"       // Parent table chunk
      }
    }
  },
  "splits": [                    // Only if split="page"
    {
      "chunks": ["chunk-id-1", "chunk-id-2"],
      "class": "page",
      "identifier": "0",
      "markdown": "string",
      "pages": [0]
    }
  ],
  "metadata": {
    "filename": "document.pdf",
    "org_id": "org_abc123",
    "page_count": 5,
    "duration_ms": 1234,
    "credit_usage": 3,
    "version": "dpt-2-latest",
    "job_id": "job_abc123",
    "failed_pages": []           // List of pages that failed to parse
  }
}
```

### 2. Extract API

**Endpoint**: `POST /extract`

Extracts structured data from documents or markdown using JSON schemas.

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `schema` | JSON string | Yes | JSON Schema defining extraction structure |
| `markdown` | string/file | One required | Markdown content or markdown file to extract from |
| `markdown_url` | string | One required | URL to markdown content |
| `model` | string | No | Model version (default: `extract-latest`) |

#### Response Structure

```json
{
  "extraction": {                // Extracted data matching schema
    "field1": "value1",
    "field2": 123,
    "nested": {
      "subfield": "value"
    },
    "array": [/* items */]
  },
  "extraction_metadata": {       // References to source chunks
    "field1": {
      "references": ["chunk-uuid-1", "chunk-uuid-2"]
    },
    "field2": {
      "references": ["chunk-uuid-3"]
    }
  },
  "metadata": {
    "credit_usage": 1,
    "duration_ms": 567,
    "filename": "document.pdf",
    "job_id": "job_xyz",
    "org_id": "org_abc",
    "version": "extract-latest",
    "fallback_model_version": null,
    "schema_violation_error": null  // Error if extraction doesn't match schema
  }
}
```

### 3. Split API

**Endpoint**: `POST /split`

Classifies and splits mixed documents by type.

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `split_class` | JSON array | Yes | Classification configuration (see below) |
| `markdown` | string | One required | Markdown content to split |
| `markdown_url` | string | One required | URL to markdown content |
| `model` | string | No | Model version (default: `split-latest`) |

#### Split Class Structure

```json
{
  "name": "Invoice",              // Required: Classification name
  "description": "Sales invoice", // Optional: Description for better classification
  "identifier": "Invoice Number"  // Optional: Field to group documents by
}
```

#### Response Structure

```json
{
  "splits": [                     // Array of classified document sections
    {
      "chunks": ["chunk-id-1", "chunk-id-2"],
      "class": "Invoice",         // Same as classification
      "classification": "Invoice",
      "identifier": "INV-001",    // Value of identifier field
      "markdowns": ["# Invoice content..."],
      "pages": [0, 1]             // Page numbers this split covers
    }
  ],
  "metadata": {
    "credit_usage": 2,
    "duration_ms": 789,
    "filename": "mixed_documents.pdf",
    "page_count": 10,
    "job_id": "job_split",
    "org_id": "org_abc",
    "version": "split-latest"
  }
}
```

### 4. Parse Jobs API (Async)

For large files (>50MB), use asynchronous processing.

#### Create Job

**Endpoint**: `POST /parse/jobs`

**Parameters**: Same as Parse API plus:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `output_save_url` | string | If ZDR | URL for zero data retention output |

**Response**:
```json
{
  "job_id": "cml1kaihb08dxcn01b3mlfy5b"
}
```

#### Get Job Status

**Endpoint**: `GET /parse/jobs/{job_id}`

**Response**:
```json
{
  "job_id": "cml1kaihb08dxcn01b3mlfy5b",
  "status": "pending|processing|completed|failed|cancelled",
  "progress": 0.75,              // 0-1 progress indicator
  "failure_reason": null,         // Error message if failed
  "received_at": 1234567890,     // Unix timestamp
  "data": { /* ParseResponse */ },   // Only when completed and output_save_url not used
  "output_url": "https://...",       // Presigned URL when result >1MB or output_save_url was set (expires 1hr)
  "org_id": "org_abc",
  "version": "dpt-2-latest",
  "metadata": { /* ParseMetadata */ }
}
```

#### List Jobs

**Endpoint**: `GET /parse/jobs`

**Query Parameters**:
- `status`: Filter by status
- `page`: Page number (0-indexed)
- `pageSize`: Items per page

**Response**:
```json
{
  "jobs": [                       // Array of job summaries
    {
      "job_id": "...",
      "status": "processing",
      "progress": 0.5,
      "failure_reason": null,
      "received_at": 1234567890
    }
  ],
  "has_more": true,
  "org_id": "org_abc"
}
```

## Data Types

### Chunk Types
- `text` - Characters, paragraphs, headings, lists, form fields, checkboxes, code blocks
- `table` - Grid of rows and columns; includes spreadsheets and receipts
- `figure` - Visual/graphical non-text content — images, graphs, flowcharts, diagrams
- `marginalia` - Content in document margins — headers, footers, page numbers, handwritten notes
- `logo` - Logos (DPT-2 only)
- `card` - ID cards and driver's licenses (DPT-2 only)
- `attestation` - Signatures, stamps, and seals (DPT-2 only)
- `scan_code` - QR codes and barcodes (DPT-2 only)

### Grounding Types

#### For Chunks (with "chunk" prefix)
- `chunkText` - Text chunk grounding
- `chunkTable` - Table chunk grounding
- `chunkFigure` - Figure chunk grounding
- `chunkMarginalia` - Marginalia chunk grounding
- `chunkLogo` - Logo chunk grounding
- `chunkCard` - Card chunk grounding
- `chunkAttestation` - Attestation chunk grounding
- `chunkScanCode` - Scan code chunk grounding

#### For Structure Elements (no prefix)
- `table` - Actual table structure
- `tableCell` - Individual table cell with position

### Bounding Box

All coordinates are normalized to 0-1 range:

```json
{
  "left": 0.1,    // Distance from left edge (10% of width)
  "top": 0.2,     // Distance from top edge (20% of height)
  "right": 0.9,   // Distance from left edge (90% of width)
  "bottom": 0.3   // Distance from top edge (30% of height)
}
```

### Table Cell Position

```json
{
  "row": 0,         // Zero-indexed row number
  "col": 0,         // Zero-indexed column number
  "rowspan": 1,     // Number of rows this cell spans
  "colspan": 2,     // Number of columns this cell spans
  "chunk_id": "..." // UUID of parent table chunk
}
```

### Table Chunk Formats

Table chunks render as HTML. The ID format and grounding availability differ by source document type.

#### PDF / Image / Document Tables

Element IDs use the format `{page_number}-{base62_sequential_number}` (page starts at 0, numbers increment per element within the page). If a page has multiple tables, numbering continues sequentially across all tables on that page. Cells may include `rowspan`/`colspan` attributes.

The `grounding` object contains bounding boxes and `tableCell` position entries for every cell.

```html
<a id='chunk-uuid'></a>

<table id="0-1">
<tr><td id="0-2" colspan="2">Product Summary</td></tr>
<tr><td id="0-3">Product</td><td id="0-4">Revenue</td></tr>
<tr><td id="0-5">Hardware</td><td id="0-6">15,230</td></tr>
</table>
```

#### Spreadsheet Tables (XLSX / CSV)

Element IDs use the format `{tab_name}-{cell_reference}` (e.g., `Sheet 1-A1`). The table element itself uses `{tab_name}-{start_cell}:{end_cell}` (e.g., `Sheet 1-A1:B4`). Embedded images and charts become `figure` chunks.

**`grounding` is `null`** for spreadsheet table chunks — cell positions are encoded in the IDs themselves.

```html
<a id='Sheet 1-A1:B4-chunk'></a>

<table id='Sheet 1-A1:B4'>
  <tr>
    <td id='Sheet 1-A1'>Program</td>
    <td id='Sheet 1-B1'>Interest Rate</td>
  </tr>
  <tr>
    <td id='Sheet 1-A2'>15 Year Fixed-Rate Mortgage</td>
    <td id='Sheet 1-B2'>0.05125</td>
  </tr>
</table>
```

## Error Responses

All errors follow this format:

```json
{
  "error": {
    "message": "Human-readable error message",
    "type": "error_type",
    "details": {
      "field": "problem_field",
      "reason": "Specific reason"
    }
  }
}
```

### HTTP Status Codes

| Status | Error Type | Description | Solution |
|--------|------------|-------------|----------|
| 400 | `validation_error` | Invalid parameters | Check request format |
| 401 | `authentication_error` | Invalid API key | Check VISION_AGENT_API_KEY |
| 413 | `payload_too_large` | File too large | Use Parse Jobs API |
| 422 | `unprocessable_entity` | Invalid file type or malformed schema | Validate file format and schema JSON |
| 429 | `rate_limit_error` | Too many requests | Implement backoff |
| 500 | `internal_error` | Server error | Retry with backoff |
| 504 | `timeout_error` | Request timeout | Use Parse Jobs API |

## Model Versions

| Operation | Current Version | Description |
|-----------|----------------|-------------|
| Parse | `dpt-2-latest` | Document parsing and OCR |
| Extract | `extract-latest` | Schema-based extraction |
| Split | `split-latest` | Document classification |

## Supported File Types

| Category | Formats | Notes |
|----------|---------|-------|
| **PDF** | PDF | Up to 100 pages; no password-protected files |
| **Images** | JPEG, JPG, PNG, APNG, BMP, DCX, DDS, DIB, GD, GIF, ICNS, JP2, PCX, PPM, PSD, TGA, TIF, TIFF, WEBP | |
| **Text Documents** | DOC, DOCX, ODT | Converted to PDF before parsing |
| **Presentations** | ODP, PPT, PPTX | Converted to PDF before parsing |
| **Spreadsheets** | CSV, XLSX | Up to 10 MB in Playground; no sheet/column/row limits |

> **Note:** Word, PowerPoint, and OpenDocument files are converted to PDF server-side before parsing. The output is the same structured markdown as a direct PDF upload.

## Best Practices

### 1. File Size Handling
- < 50MB: Use synchronous Parse API
- > 50MB: Use Parse Jobs API
- > 100MB: Consider splitting document first

### 2. Rate Limiting
- Implement exponential backoff
- Start with 1 second delay
- Double delay on each retry
- Maximum 5 retries

### 3. Schema Design
- Keep schemas simple and flat when possible
- Use clear descriptions for each field
- Mark optional fields appropriately
- Test schema with sample data first

### 4. Cost Optimization
- Parse once, extract multiple times
- Use specific schemas (avoid extracting everything)
- Cache parsed results when possible

---

# API (curl) Reference

Direct HTTP API implementation using curl and shell scripts.

## Authentication

```bash
export VISION_AGENT_API_KEY="v2_..."
# All requests need:
-H "Authorization: Bearer $VISION_AGENT_API_KEY"
```

## 1. Parse Examples

> See [Parse API Specification](#1-parse-api) for parameters and response structure.

### Basic Parse
```bash
curl -X POST https://api.landing.ai/v1/ade/parse \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document=@document.pdf" \
  -F "model=dpt-2-latest"
```

### Parse with Page Splitting
```bash
curl -X POST https://api.landing.ai/v1/ade/parse \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document=@multi_page.pdf" \
  -F "split=page"
```

### Parse from URL
```bash
curl -X POST https://api.landing.ai/v1/ade/parse \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document_url=https://example.com/document.pdf"
```

## 2. Extract Examples

> See [Extract API Specification](#2-extract-api) for parameters and response structure.

### Extract from Markdown File
```bash
SCHEMA='{
  "type": "object",
  "properties": {
    "invoice_number": {"type": "string", "description": "Invoice number"},
    "total_amount": {"type": "number", "description": "Total amount"},
    "vendor_name": {"type": "string", "description": "Vendor name"}
  }
}'

# Extract accepts markdown, not raw documents — parse first if needed
curl -X POST https://api.landing.ai/v1/ade/extract \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "markdown=@parsed_invoice.md" \
  -F "schema=$SCHEMA" \
  -F "model=extract-latest"
```

### Extract from Parsed Markdown (Parse Once, Extract Many)
```bash
# Parse once
MARKDOWN=$(curl -s -X POST https://api.landing.ai/v1/ade/parse \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document=@invoice.pdf" \
  | jq -r '.markdown')

# Extract with different schemas
curl -X POST https://api.landing.ai/v1/ade/extract \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "markdown=$MARKDOWN" \
  -F "schema=$SCHEMA"
```

### Complex Nested Extraction
```bash
PO_SCHEMA='{
  "type": "object",
  "properties": {
    "po_number": {"type": "string"},
    "line_items": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "sku": {"type": "string"},
          "quantity": {"type": "integer"},
          "unit_price": {"type": "number"}
        }
      }
    },
    "total": {"type": "number"}
  }
}'

# Parse first, then extract
MARKDOWN=$(curl -s -X POST https://api.landing.ai/v1/ade/parse \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document=@purchase_order.pdf" \
  | jq -r '.markdown')

curl -X POST https://api.landing.ai/v1/ade/extract \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "markdown=$MARKDOWN" \
  -F "schema=$PO_SCHEMA"
```

## 3. Split Examples

> See [Split API Specification](#3-split-api) for parameters and response structure.

### Basic Document Splitting
```bash
SPLIT_CLASSES='[
  {"name": "Invoice", "identifier": "Invoice Number"},
  {"name": "Receipt", "identifier": "Receipt Number"},
  {"name": "Purchase Order", "identifier": "PO Number"}
]'

# Parse first, then split
MARKDOWN=$(curl -s -X POST https://api.landing.ai/v1/ade/parse \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document=@mixed_documents.pdf" \
  | jq -r '.markdown')

curl -X POST https://api.landing.ai/v1/ade/split \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "markdown=$MARKDOWN" \
  -F "split_class=$SPLIT_CLASSES" \
  -F "model=split-latest"
```

## 4. Parse Jobs (Async, Large Files)

> See [Parse Jobs Specification](#4-parse-jobs-api-async) for parameters and response structure.

### Create and Monitor Job
```bash
#!/bin/bash

# Create job for large file
JOB_ID=$(curl -s -X POST https://api.landing.ai/v1/ade/parse/jobs \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document=@large_document.pdf" \
  -F "model=dpt-2-latest" \
  | jq -r '.job_id')

echo "Created job: $JOB_ID"

# Poll for completion
while true; do
  STATUS=$(curl -s -X GET "https://api.landing.ai/v1/ade/parse/jobs/$JOB_ID" \
    -H "Authorization: Bearer $VISION_AGENT_API_KEY")

  STATE=$(echo "$STATUS" | jq -r '.status')
  PROGRESS=$(echo "$STATUS" | jq -r '.progress')

  echo "Status: $STATE, Progress: $(echo "$PROGRESS * 100" | bc)%"

  if [ "$STATE" = "completed" ]; then
    echo "$STATUS" | jq '.data' > "parse_result.json"
    break
  elif [ "$STATE" = "failed" ]; then
    echo "Job failed: $(echo "$STATUS" | jq -r '.failure_reason')" >&2
    exit 1
  fi

  sleep 5
done
```

## Complete Workflows

### Parse, Split, and Extract Pipeline
```bash
#!/bin/bash

# 1. Parse mixed document
PARSED=$(curl -s -X POST https://api.landing.ai/v1/ade/parse \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document=@mixed_invoices.pdf")

MARKDOWN=$(echo "$PARSED" | jq -r '.markdown')

# 2. Split by document type
SPLIT_CLASSES='[
  {"name": "Invoice", "identifier": "Invoice Number"},
  {"name": "Credit Note", "identifier": "Credit Note Number"}
]'

SPLITS=$(curl -s -X POST https://api.landing.ai/v1/ade/split \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "markdown=$MARKDOWN" \
  -F "split_class=$SPLIT_CLASSES")

# 3. Extract from each split
SCHEMA='{"type": "object", "properties": {
  "document_number": {"type": "string"},
  "total": {"type": "number"},
  "date": {"type": "string"}
}}'

echo "$SPLITS" | jq -c '.splits[]' | while read -r split; do
  TYPE=$(echo "$split" | jq -r '.classification')
  ID=$(echo "$split" | jq -r '.identifier')
  MD=$(echo "$split" | jq -r '.markdowns[0]')

  echo "Processing $TYPE: $ID"

  curl -s -X POST https://api.landing.ai/v1/ade/extract \
    -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
    -F "markdown=$MD" \
    -F "schema=$SCHEMA" \
    | jq '.extraction'
done
```

### Table Data Extraction
```bash
#!/bin/bash

PARSED=$(curl -s -X POST https://api.landing.ai/v1/ade/parse \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document=@financial_report.pdf")

# Get first table's markdown
TABLE_MD=$(echo "$PARSED" | jq -r '
  .chunks[] | select(.type == "table") | .markdown' | head -1)

if [ -z "$TABLE_MD" ]; then
  echo "No tables found" >&2
  exit 1
fi

TABLE_SCHEMA='{"type": "object", "properties": {
  "revenue_2023": {"type": "number"},
  "revenue_2024": {"type": "number"},
  "profit_2023": {"type": "number"},
  "profit_2024": {"type": "number"}
}}'

curl -s -X POST https://api.landing.ai/v1/ade/extract \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "markdown=$TABLE_MD" \
  -F "schema=$TABLE_SCHEMA" \
  | jq '.extraction'
```

## Error Handling

### HTTP Status Check with Retry
```bash
#!/bin/bash

MAX_RETRIES=3
RETRY_COUNT=0

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
  RESPONSE=$(curl -s -w "\n%{http_code}" -X POST https://api.landing.ai/v1/ade/parse \
    -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
    -F "document=@document.pdf")

  HTTP_CODE=$(echo "$RESPONSE" | tail -n 1)
  BODY=$(echo "$RESPONSE" | sed '$d')

  if [ "$HTTP_CODE" -eq 200 ]; then
    echo "$BODY"
    break
  elif [ "$HTTP_CODE" -eq 429 ]; then
    WAIT_TIME=$((2 ** RETRY_COUNT * 10))
    echo "Rate limited. Waiting ${WAIT_TIME}s..." >&2
    sleep $WAIT_TIME
    RETRY_COUNT=$((RETRY_COUNT + 1))
  elif [ "$HTTP_CODE" -eq 413 ] || [ "$HTTP_CODE" -eq 504 ]; then
    echo "File too large or timeout — use parse jobs API" >&2
    exit 1
  else
    echo "Error: HTTP $HTTP_CODE" >&2
    echo "$BODY" | jq '.error' >&2
    exit 1
  fi
done
```

## jq Recipes

```bash
# Extract just markdown
curl -s ... | jq -r '.markdown'

# Get all tables
curl -s ... | jq '.chunks[] | select(.type == "table")'

# Extract table cells with positions
curl -s ... | jq '.grounding | to_entries[] | select(.value.type == "tableCell")'

# Get chunks from specific page
curl -s ... | jq '.chunks[] | select(.grounding.page == 0)'

# Group chunks by type with counts
curl -s ... | jq '.chunks | group_by(.type) | map({type: .[0].type, count: length})'

# Tables as CSV
curl -s ... | jq -r '.chunks[] | select(.type == "table") | .markdown' | sed 's/|/,/g; s/^,//; s/,$//'

# Calculate bounding box areas
curl -s ... | jq '[.chunks[] | .grounding.box | ((.right - .left) * (.bottom - .top))] | add'

# Get specific extracted field
curl -s ... | jq '.extraction.invoice_number'

# Process extracted line items
curl -s ... | jq '.extraction.line_items[] | {sku: .sku, total: (.quantity * .unit_price)}'
```

## Best Practices

### Save Intermediate Results
```bash
# Save parsed output for reuse (parse once, extract many)
curl -s -X POST https://api.landing.ai/v1/ade/parse \
  -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
  -F "document=@document.pdf" | tee parsed_output.json

# Later, extract from saved markdown
MARKDOWN=$(jq -r '.markdown' < parsed_output.json)
```

### Shell Functions for Reuse
```bash
ade_parse() {
  curl -s -X POST https://api.landing.ai/v1/ade/parse \
    -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
    -F "document=@$1"
}

ade_extract() {
  curl -s -X POST https://api.landing.ai/v1/ade/extract \
    -H "Authorization: Bearer $VISION_AGENT_API_KEY" \
    -F "markdown=$1" \
    -F "schema=$2"
}
```
