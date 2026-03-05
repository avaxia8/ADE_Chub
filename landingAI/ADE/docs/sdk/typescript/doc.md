---
name: landingai-ade-typescript
description: "TypeScript/JavaScript SDK reference for LandingAI's Agentic Document Extraction (ADE). Includes type definitions, Zod schema validation, async processing, error handling, type guards, and complete API context."
languages: typescript/javascript
versions: ">=0.1.0"
updated-on: 2026-03-04
source: maintainer
tags: landingai,ade,typescript,javascript,sdk,zod,document-extraction,parse,extract,split,async
---

# LandingAI ADE — TypeScript SDK Reference

TypeScript/JavaScript SDK for LandingAI's Agentic Document Extraction.

## Installation

```bash
npm install landingai-ade
# or: yarn add landingai-ade / pnpm add landingai-ade
export VISION_AGENT_API_KEY="v2_..."
```

## Client Setup

```typescript
import { LandingAIADE } from "landingai-ade";

const client = new LandingAIADE();  // Uses VISION_AGENT_API_KEY env var

// Or pass key directly
const client = new LandingAIADE({ apiKey: "v2_..." });

// EU region
const client = new LandingAIADE({
  baseUrl: "https://api.va.eu-west-1.landing.ai/v1/ade"
});

// Full config
const client = new LandingAIADE({
  apiKey: "v2_...",
  timeout: 60000,
  maxRetries: 3,
});
```

## Type Definitions

```typescript
interface ParseResponse {
  markdown: string;
  chunks: Chunk[];
  grounding: Record<string, Grounding>;
  splits?: Split[];
  metadata: Metadata;
}

interface Chunk {
  id: string;
  type: "text" | "table" | "figure" | "marginalia" | "logo" | "card" | "attestation" | "scan_code";
  markdown: string;
  grounding: { page: number; box: BoundingBox };
}

interface BoundingBox {
  left: number; top: number; right: number; bottom: number;  // 0-1 normalized
}

interface Grounding {
  type: string;
  page: number;
  box: BoundingBox;
  position?: TablePosition;  // Only for tableCell type
}

interface TablePosition {
  row: number; col: number; rowspan: number; colspan: number; chunk_id: string;
}

interface ExtractResponse {
  extraction: Record<string, any>;
  extraction_metadata: Record<string, { references?: string[] }>;
  metadata: Metadata;
}

interface SplitResponse {
  splits: Split[];
  metadata: Metadata;
}

interface Split {
  chunks: string[];
  class: string;
  classification: string;
  identifier: string;
  markdowns: string[];
  pages: number[];
}

interface Metadata {
  filename: string; org_id: string; page_count: number;
  duration_ms: number; credit_usage: number; version: string;
  job_id: string; failed_pages?: number[];
}
```

## 1. Parse API

### Function Signature
```typescript
async parse(options: {
  document?: string | Buffer | Readable;
  documentUrl?: string;
  model?: string;       // Default: "dpt-2-latest"
  split?: "page";
  saveTo?: string;
}): Promise<ParseResponse>
```

### Basic Usage
```typescript
// Parse from file path
const response = await client.parse({ document: "./invoice.pdf" });
console.log(response.markdown);
console.log(response.chunks.length);

// Parse from URL
const response = await client.parse({
  documentUrl: "https://example.com/document.pdf"
});

// Parse from Buffer
import * as fs from "fs";
const buffer = fs.readFileSync("./invoice.pdf");
const response = await client.parse({ document: buffer });
```

### Page Splitting
```typescript
const response = await client.parse({
  document: "./multi_page.pdf",
  split: "page"
});

response.splits?.forEach((split, idx) => {
  console.log(`Page ${idx + 1}: ${split.chunks.length} chunks`);
});
```

### Working with Chunks and Grounding
```typescript
const response = await client.parse({ document: "./document.pdf" });

// Filter by type
const tables = response.chunks.filter(c => c.type === "table");
const page0 = response.chunks.filter(c => c.grounding.page === 0);

// Find table cells
const tableCells = Object.entries(response.grounding)
  .filter(([_, g]) => g.type === "tableCell")
  .map(([id, g]) => ({ id, page: g.page, position: g.position! }));

tableCells.forEach(cell => {
  const { row, col, rowspan, colspan } = cell.position;
  console.log(`Cell (${row},${col}) span ${rowspan}x${colspan}`);
});
```

### Save Output
```typescript
await client.parse({
  document: "./document.pdf",
  saveTo: "./output"
});
// Creates: output/document_parse_output.json, output/document.md
```

## 2. Extract API

### Function Signature
```typescript
async extract(options: {
  schema: string;                          // JSON Schema as string
  markdown?: string;                       // Markdown content or file path
  markdownUrl?: string;
  model?: string;       // Default: "extract-latest"
  saveTo?: string;
}): Promise<ExtractResponse>
```

### Basic Extraction
```typescript
const schema = {
  type: "object",
  properties: {
    invoice_number: { type: "string", description: "Invoice number" },
    total_amount: { type: "number", description: "Total amount" },
    vendor_name: { type: "string", description: "Vendor name" },
  },
  required: ["invoice_number", "total_amount"]
};

// Direct from markdown file
const response = await client.extract({
  markdown: "./invoice_parsed.md",
  schema: JSON.stringify(schema),
  model: "extract-latest"
});

console.log(response.extraction.invoice_number);
console.log(response.extraction.total_amount);
```

### Extract from Parsed Markdown (Parse Once, Extract Many)
```typescript
const parsed = await client.parse({ document: "./document.pdf" });

const [header, financial] = await Promise.all([
  client.extract({
    markdown: parsed.markdown,
    schema: JSON.stringify(headerSchema),
  }),
  client.extract({
    markdown: parsed.markdown,
    schema: JSON.stringify(financialSchema),
  }),
]);
```

### Using Zod for Schema Validation
```typescript
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";

const InvoiceSchema = z.object({
  invoice_number: z.string().describe("Invoice number or ID"),
  total_amount: z.number().positive().describe("Total amount"),
  vendor_name: z.string().describe("Vendor name"),
  line_items: z.array(z.object({
    description: z.string(),
    quantity: z.number().int().positive(),
    unit_price: z.number().positive(),
    total: z.number().positive()
  })).optional()
});

const response = await client.extract({
  document: "./invoice.pdf",
  schema: JSON.stringify(zodToJsonSchema(InvoiceSchema)),
});

// Validate extracted data
const validated = InvoiceSchema.parse(response.extraction);
```

### Grounding References (Tracing Back to Source)
```typescript
const parsed = await client.parse({ document: "./document.pdf" });
const chunkMap = new Map(parsed.chunks.map(c => [c.id, c]));

const response = await client.extract({
  markdown: parsed.markdown,
  schema: JSON.stringify(schema),
});

Object.entries(response.extraction).forEach(([field, value]) => {
  const refs = response.extraction_metadata[field]?.references;
  if (refs?.length) {
    const chunk = chunkMap.get(refs[0]);
    if (chunk) {
      console.log(`${field}=${value} → page ${chunk.grounding.page}`);
    }
  }
});
```

### Extract from Table Chunks
```typescript
const parsed = await client.parse({ document: "./report.pdf" });
const tables = parsed.chunks.filter(c => c.type === "table");

if (tables.length > 0) {
  const response = await client.extract({
    markdown: tables[0].markdown,
    schema: JSON.stringify({
      type: "object",
      properties: {
        revenue: { type: "number", description: "Total revenue" },
        expenses: { type: "number", description: "Total expenses" }
      }
    }),
  });
  console.log(response.extraction);
}
```

## 3. Split API

### Function Signature
```typescript
async split(options: {
  splitClass: Array<{ name: string; description?: string; identifier?: string }>;
  markdown?: string;
  markdownUrl?: string;
  model?: string;  // Default: "split-latest"
}): Promise<SplitResponse>
```

### Basic Splitting
```typescript
const parsed = await client.parse({ document: "./mixed_documents.pdf" });

const response = await client.split({
  markdown: parsed.markdown,
  splitClass: [
    { name: "Invoice", description: "Sales invoice", identifier: "Invoice Number" },
    { name: "Receipt", description: "Payment receipt", identifier: "Receipt Number" },
  ],
  model: "split-latest"
});

response.splits.forEach(split => {
  console.log(`${split.classification}: ${split.identifier} (pages ${split.pages})`);
});
```

### Split → Extract Workflow
```typescript
async function splitAndExtract(client: LandingAIADE, documentPath: string) {
  const parsed = await client.parse({ document: documentPath });

  const splitResponse = await client.split({
    markdown: parsed.markdown,
    splitClass: [
      { name: "Invoice", identifier: "Invoice Number" },
      { name: "Credit Note", identifier: "Credit Note Number" }
    ],
  });

  const schema = JSON.stringify({
    type: "object",
    properties: {
      document_number: { type: "string" },
      total: { type: "number" },
      date: { type: "string" },
    }
  });

  const results = [];
  for (const split of splitResponse.splits) {
    const extracted = await client.extract({
      markdown: split.markdowns[0],
      schema,
    });
    results.push({
      type: split.classification,
      id: split.identifier,
      data: extracted.extraction
    });
  }
  return results;
}
```

## 4. Parse Jobs (Async, Large Files)

### Function Signatures
```typescript
// client.parseJobs
async create(options: {
  document?: string | Buffer | Readable;
  documentUrl?: string;
  model?: string;
  split?: "page";
  outputSaveUrl?: string;  // For ZDR
}): Promise<{ job_id: string }>

async get(jobId: string): Promise<{
  job_id: string;
  status: "pending" | "processing" | "completed" | "failed";
  progress: number;
  failure_reason?: string;
  data?: ParseResponse;          // Present when completed and output_save_url not used
  output_url?: string;           // Presigned URL when result >1MB or output_save_url was set
}>

async list(options?: {
  status?: string; page?: number; pageSize?: number;
}): Promise<{ jobs: JobSummary[]; has_more: boolean }>
```

### Create and Monitor Job
```typescript
async function parseLargeFile(
  client: LandingAIADE, filePath: string
): Promise<ParseResponse> {
  const job = await client.parseJobs.create({
    document: filePath,
    model: "dpt-2-latest"
  });

  while (true) {
    const status = await client.parseJobs.get(job.job_id);
    console.log(`${status.status}: ${(status.progress * 100).toFixed(0)}%`);

    if (status.status === "completed") return status.data!;
    if (status.status === "failed") {
      throw new Error(`Job failed: ${status.failure_reason}`);
    }

    await new Promise(r => setTimeout(r, 5000));
  }
}
```

### Auto-Detect File Size
```typescript
import * as fs from "fs";

async function parseAuto(client: LandingAIADE, filePath: string) {
  const sizeMB = fs.statSync(filePath).size / (1024 * 1024);

  if (sizeMB > 50) {
    console.log(`${sizeMB.toFixed(1)}MB — using async jobs`);
    return await parseLargeFile(client, filePath);
  }

  return await client.parse({ document: filePath });
}
```

## Error Handling

### Error Types
```typescript
import {
  LandingAIADEError,      // Base error class
  APIConnectionError,     // Network errors
  APITimeoutError,        // Request timeout
  APIStatusError,         // HTTP status errors (has .status)
  RateLimitError,         // 429 rate limit
  AuthenticationError,    // 401 unauthorized
  BadRequestError,        // 400 bad request
} from "landingai-ade/errors";
```

### Retry with Fallback to Jobs
```typescript
async function robustParse(
  client: LandingAIADE, filePath: string, maxRetries = 3
): Promise<ParseResponse> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await client.parse({ document: filePath });
    } catch (error) {
      if (error instanceof RateLimitError) {
        const wait = Math.pow(2, attempt) * 10000;
        console.log(`Rate limited, waiting ${wait}ms...`);
        await new Promise(r => setTimeout(r, wait));
      } else if (error instanceof APITimeoutError) {
        console.log("Timeout — switching to parse jobs");
        return await parseLargeFile(client, filePath);
      } else if (error instanceof AuthenticationError) {
        throw error;  // Non-retryable
      } else if (error instanceof APIConnectionError) {
        await new Promise(r => setTimeout(r, 2000));
      } else {
        throw error;
      }
    }
  }
  throw new Error("Failed after retries");
}
```

## Type Guards

```typescript
function isTableChunk(chunk: Chunk): boolean {
  return chunk.type === "table";
}

function isTableCell(
  grounding: Grounding
): grounding is Grounding & { position: TablePosition } {
  return grounding.type === "tableCell" && grounding.position !== undefined;
}

// Usage
Object.values(response.grounding).forEach(g => {
  if (isTableCell(g)) {
    console.log(`Cell at (${g.position.row}, ${g.position.col})`);
  }
});
```

---

## API Reference

The following sections provide the complete API context so this document is fully self-contained.

### Base Configuration

| Region | Base URL |
|--------|----------|
| US (default) | `https://api.va.landing.ai/v1/ade` |
| EU | `https://api.va.eu-west-1.landing.ai/v1/ade` |

**Authentication**: All requests require `Authorization: Bearer $VISION_AGENT_API_KEY`

### API Endpoints Summary

| Endpoint | Method | Path | Model | Input |
|----------|--------|------|-------|-------|
| Parse | POST | `/v1/ade/parse` | `dpt-2-latest` | `document` (file) or `document_url` |
| Extract | POST | `/v1/ade/extract` | `extract-latest` | `markdown` (file/string) or `markdown_url` + `schema` |
| Split | POST | `/v1/ade/split` | `split-latest` | `markdown` (file/string) or `markdown_url` + `split_class` |
| Create Job | POST | `/v1/ade/parse/jobs` | `dpt-2-latest` | `document` or `document_url` |
| Get Job | GET | `/v1/ade/parse/jobs/{id}` | — | — |
| List Jobs | GET | `/v1/ade/parse/jobs` | — | `?status=&page=&pageSize=` |

### Parse API — Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `document` | file | One required | Local file — PDF, images (JPG/PNG/TIFF/WEBP/GIF/BMP/PSD + more), Word (DOC/DOCX/ODT), PowerPoint (PPT/PPTX/ODP), spreadsheets (XLSX/CSV) |
| `document_url` | string | One required | Remote document URL |
| `model` | string | No | Model version (default: `dpt-2-latest`) |
| `split` | string | No | Split mode: `"page"` to split by pages |

### Parse API — Response Structure

```json
{
  "markdown": "string",
  "chunks": [
    {
      "id": "uuid",
      "type": "text|table|marginalia|figure|scan_code|logo|card|attestation",
      "markdown": "string",
      "grounding": {
        "page": 0,
        "box": { "left": 0.1, "top": 0.2, "right": 0.9, "bottom": 0.3 }
      }
    }
  ],
  "grounding": {
    "chunk-id": {
      "type": "chunkText|chunkTable|chunkFigure|...",
      "page": 0,
      "box": { "left": 0.1, "top": 0.2, "right": 0.9, "bottom": 0.3 }
    },
    "0-1": { "type": "table", "page": 0, "box": { } },
    "0-2": {
      "type": "tableCell", "page": 0, "box": { },
      "position": { "row": 0, "col": 0, "rowspan": 1, "colspan": 1, "chunk_id": "uuid" }
    }
  },
  "splits": [
    { "chunks": ["id1"], "class": "page", "identifier": "0", "markdown": "string", "pages": [0] }
  ],
  "metadata": {
    "filename": "document.pdf", "org_id": "org_abc", "page_count": 5,
    "duration_ms": 1234, "credit_usage": 3, "version": "dpt-2-latest",
    "job_id": "job_abc", "failed_pages": []
  }
}
```

### Extract API — Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `schema` | JSON string | Yes | JSON Schema defining extraction structure |
| `markdown` | string/file | One required | Markdown content or markdown file to extract from |
| `markdown_url` | string | One required | URL to markdown content |
| `model` | string | No | Model version (default: `extract-latest`) |

### Extract API — Response Structure

```json
{
  "extraction": { "field1": "value1", "field2": 123 },
  "extraction_metadata": {
    "field1": { "references": ["chunk-uuid-1", "chunk-uuid-2"] }
  },
  "metadata": {
    "credit_usage": 1, "duration_ms": 567, "filename": "document.pdf",
    "job_id": "job_xyz", "org_id": "org_abc", "version": "extract-latest",
    "fallback_model_version": null, "schema_violation_error": null
  }
}
```

### Split API — Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `split_class` | JSON array | Yes | Classification configuration |
| `markdown` | string | One required | Markdown content to split |
| `markdownUrl` | string | One required | URL to markdown content |
| `model` | string | No | Model version (default: `split-latest`) |

### Split API — Response Structure

```json
{
  "splits": [
    {
      "chunks": ["chunk-id-1"], "class": "Invoice", "classification": "Invoice",
      "identifier": "INV-001", "markdowns": ["# Invoice content..."], "pages": [0, 1]
    }
  ],
  "metadata": {
    "credit_usage": 2, "duration_ms": 789, "filename": "mixed.pdf",
    "page_count": 10, "job_id": "job_split", "org_id": "org_abc", "version": "split-latest"
  }
}
```

### Parse Jobs API

**Create**: `POST /parse/jobs` — same parameters as Parse plus optional `output_save_url` for ZDR.

**Get Status**: `GET /parse/jobs/{job_id}` — returns `{ job_id, status, progress (0-1), failure_reason, data (ParseResponse when completed), output_url (presigned, expires 1hr) }`.

**List**: `GET /parse/jobs?status=&page=&pageSize=` — returns `{ jobs: [{ job_id, status, progress, received_at }], has_more }`.

### Data Types

#### Chunk Types
- `text` — Characters, paragraphs, headings, lists, form fields, checkboxes, code blocks
- `table` — Grid of rows and columns; includes spreadsheets and receipts
- `figure` — Visual/graphical non-text content — images, graphs, flowcharts, diagrams
- `marginalia` — Content in document margins — headers, footers, page numbers, handwritten notes
- `logo` — Logos (DPT-2 only)
- `card` — ID cards and driver's licenses (DPT-2 only)
- `attestation` — Signatures, stamps, and seals (DPT-2 only)
- `scan_code` — QR codes and barcodes (DPT-2 only)

#### Grounding Types
- Chunk grounding: `chunkText`, `chunkTable`, `chunkFigure`, `chunkMarginalia`, `chunkLogo`, `chunkCard`, `chunkAttestation`, `chunkScanCode`
- Structure: `table`, `tableCell` (with position data)

#### Bounding Box
All coordinates normalized 0–1: `{ left, top, right, bottom }`.

#### Table Cell Position
`{ row, col, rowspan, colspan, chunk_id }` — zero-indexed.

#### Table Chunk Formats

**PDF/Image tables**: Element IDs use `{page}-{base62_seq}`. Grounding object has bounding boxes and `tableCell` entries.

**Spreadsheet tables (XLSX/CSV)**: Element IDs use `{tab_name}-{cell_ref}` (e.g., `Sheet 1-B2`). **Grounding is null** — positions are encoded in IDs.

### Error Codes

| Status | Error Type | Description | Solution |
|--------|------------|-------------|----------|
| 400 | `validation_error` | Invalid parameters | Check request format |
| 401 | `authentication_error` | Invalid API key | Check VISION_AGENT_API_KEY |
| 413 | `payload_too_large` | File too large | Use Parse Jobs API |
| 422 | `unprocessable_entity` | Invalid file type or malformed schema | Validate file format and schema JSON |
| 429 | `rate_limit_error` | Too many requests | Implement backoff |
| 500 | `internal_error` | Server error | Retry with backoff |
| 504 | `timeout_error` | Request timeout | Use Parse Jobs API |

### Supported File Types

| Category | Formats | Notes |
|----------|---------|-------|
| **PDF** | PDF | Up to 100 pages; no password-protected files |
| **Images** | JPEG, JPG, PNG, APNG, BMP, DCX, DDS, DIB, GD, GIF, ICNS, JP2, PCX, PPM, PSD, TGA, TIF, TIFF, WEBP | |
| **Text Documents** | DOC, DOCX, ODT | Converted to PDF before parsing |
| **Presentations** | ODP, PPT, PPTX | Converted to PDF before parsing |
| **Spreadsheets** | CSV, XLSX | Up to 10 MB in Playground; no sheet/column/row limits |

> **Note:** Word, PowerPoint, and OpenDocument files are converted to PDF server-side before parsing.

### Model Versions

| Operation | Current Version | Description |
|-----------|----------------|-------------|
| Parse | `dpt-2-latest` | Document parsing and OCR |
| Extract | `extract-latest` | Schema-based extraction |
| Split | `split-latest` | Document classification |
