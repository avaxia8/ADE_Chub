# LandingAI ADE — TypeScript SDK Reference

TypeScript/JavaScript SDK for LandingAI's Agentic Document Extraction.

> For response structures, data types, and error codes see the [API Specification](ADE_API.md).

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
  type: "text" | "table" | "figure" | "formula" | "list";
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

> See [Parse API Specification](ADE_API.md#1-parse-api) for request parameters and full response details.

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

> See [Extract API Specification](ADE_API.md#2-extract-api) for request parameters and full response details.

### Function Signature
```typescript
async extract(options: {
  schema: string;                          // JSON Schema as string
  document?: string | Buffer | Readable;
  markdown?: string;
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

// Direct from document
const response = await client.extract({
  document: "./invoice.pdf",
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

> See [Split API Specification](ADE_API.md#3-split-api) for request parameters and full response details.

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

> See [Parse Jobs Specification](ADE_API.md#4-parse-jobs-api-async) for parameters and response structure.

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
  result?: ParseResponse;
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

    if (status.status === "completed") return status.result!;
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
