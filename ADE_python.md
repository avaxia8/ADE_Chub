# LandingAI ADE — Python SDK Reference

Python SDK for LandingAI's Agentic Document Extraction.

> For response structures, data types, and error codes see the [API Specification](ADE_API.md).

## Installation

```bash
pip install landingai-ade
export VISION_AGENT_API_KEY="v2_..."
```

## Client Setup

```python
from landingai_ade import LandingAIADE
from pathlib import Path

client = LandingAIADE()  # Uses VISION_AGENT_API_KEY env var

# Or pass key directly
client = LandingAIADE(api_key="v2_...")

# EU region
client = LandingAIADE(base_url="https://api.va.eu-west-1.landing.ai/v1/ade")
```

## 1. Parse API

> See [Parse API Specification](ADE_API.md#1-parse-api) for request parameters and response structure.

### Basic Usage
```python
response = client.parse(
    # use document= for local files, document_url= for remote URLs
    document=Path("path/to/file"),
    model="dpt-2-latest",
    save_to="./output_folder",  # optional: saves as {input_file}_parse_output.json
)

print(response.markdown)           # Full document text
print(len(response.chunks))        # Number of content blocks
print(response.metadata.page_count)  # Page count

# Filter chunks by type
tables = [c for c in response.chunks if c.type == "table"]
text = [c for c in response.chunks if c.type == "text"]
```

### Parse Options
```python
# From URL
response = client.parse(document_url="https://example.com/doc.pdf")

# With page splitting
response = client.parse(document=Path("multi_page.pdf"), split="page")
for split in response.splits:
    print(f"Page {split.pages}: {len(split.chunks)} chunks")

# Save output to directory
response = client.parse(
    document=Path("doc.pdf"),
    save_to="./output",  # saves as {input_file}_parse_output.json
)
```

### Visual Grounding and Table Cells
```python
response = client.parse(document=Path("doc.pdf"))

# Access chunk locations
for chunk in response.chunks:
    box = chunk.grounding.box
    print(f"{chunk.type} on page {chunk.grounding.page}: "
          f"({box.left:.3f}, {box.top:.3f}) → ({box.right:.3f}, {box.bottom:.3f})")

# Find table cells with positions
for gid, grounding in response.grounding.items():
    if grounding.type == "tableCell":
        pos = grounding.position
        print(f"Cell ({pos.row}, {pos.col}) span=({pos.rowspan}x{pos.colspan})")
```

### Extract a Cell Value by Row and Column (PDF)
```python
import re

response = client.parse(document=Path("doc.pdf"))

# Get the first table chunk
table = next(c for c in response.chunks if c.type == "table")

# Parse HTML rows and cells into a (row, col) grid.
# Table cell content lives in the chunk markdown as HTML.
rows = re.findall(r'<tr[^>]*>(.*?)</tr>', table.markdown, re.DOTALL)
grid = {}
for r, row_html in enumerate(rows):
    for c, m in enumerate(re.finditer(
        r'<td[^>]*>(.*?)</td>', row_html, re.DOTALL
    )):
        grid[(r, c)] = re.sub(r'<[^>]+>', '', m.group(1)).strip()

# Look up a value (zero-indexed row and column)
row, col = 1, 0
value = grid[(row, col)]
print(f"Row {row}, Col {col}: {value}")
```

### Read a Spreadsheet Cell by Reference
```python
import re

response = client.parse(document=Path("report.xlsx"))

# Get the first table chunk
table = next(c for c in response.chunks if c.type == "table")

# Spreadsheet cell IDs are "{tab_name}-{cell_ref}" (e.g., "Sheet 1-B2").
# grounding is null for spreadsheets, so parse IDs directly from HTML.
cell_text = {}
for m in re.finditer(
    r'<td[^>]*\bid=["\']([^"\']+)["\'][^>]*>(.*?)</td>',
    table.markdown,
    re.DOTALL,
):
    cell_text[m.group(1)] = re.sub(r"<[^>]+>", "", m.group(2)).strip()

# Look up by tab name and cell reference
value = cell_text["Sheet 1-B2"]
print(f"Sheet 1, cell B2: {value}")
```

## 2. Extract API

> See [Extract API Specification](ADE_API.md#2-extract-api) for request parameters and response structure.

### Pydantic Schema Extraction
```python
from pydantic import BaseModel, Field
from landingai_ade.lib import pydantic_to_json_schema

class InvoiceData(BaseModel):
    invoice_number: str = Field(description="Invoice number or ID")
    total_amount: float = Field(description="Total amount to be paid")
    vendor_name: str = Field(description="Vendor or supplier name")
    line_items: list[dict] | None = Field(default=None, description="Line items")

# Parse then extract (recommended: parse once, extract many)
parsed = client.parse(document=Path("invoice.pdf"))

response = client.extract(
    markdown=parsed.markdown,
    schema=pydantic_to_json_schema(InvoiceData),
    model="extract-latest"
)

invoice = InvoiceData(**response.extraction)
print(f"Invoice {invoice.invoice_number}: ${invoice.total_amount}")
```

### Direct Extraction from File
```python
response = client.extract(
    schema=pydantic_to_json_schema(InvoiceData),
    # use markdown= for local files, markdown_url= for remote URLs
    markdown=Path("path/to/file.md"),
    save_to="./output_folder",  # optional: saves as {input_file}_extract_output.json
)
```

### Grounding References (Tracing Back to Source)
```python
parsed = client.parse(document=Path("doc.pdf"))
chunk_map = {c.id: c for c in parsed.chunks}

response = client.extract(
    markdown=parsed.markdown,
    schema=pydantic_to_json_schema(InvoiceData)
)

for field, meta in response.extraction_metadata.items():
    if meta.get("references"):
        chunk = chunk_map.get(meta["references"][0])
        if chunk:
            print(f"{field}: page {chunk.grounding.page}, type={chunk.type}")
```

## 3. Split API

> See [Split API Specification](ADE_API.md#3-split-api) for request parameters and response structure.

### Basic Splitting
```python
parsed = client.parse(document=Path("mixed_documents.pdf"))

split_class = [
    {"name": "Invoice", "description": "Sales invoice", "identifier": "Invoice Number"},
    {"name": "Receipt", "description": "Payment receipt", "identifier": "Receipt Number"},
]

split_response = client.split(
    split_class=split_class,
    markdown=parsed.markdown,  # Pass Markdown string directly
    model="split-latest",
    save_to="output_folder",  # optional: saves as {input_file}_split_output.json
)

for split in split_response.splits:
    print(f"{split.classification}: {split.identifier} (pages {split.pages})")
```

### Split → Extract Pipeline
```python
parsed = client.parse(document=Path("mixed_invoices.pdf"))

splits = client.split(
    markdown=parsed.markdown,
    split_class=[
        {"name": "Invoice", "identifier": "Invoice Number"},
        {"name": "Credit Note", "identifier": "Credit Note Number"},
    ]
)

schema = json.dumps({
    "type": "object",
    "properties": {
        "document_number": {"type": "string"},
        "total": {"type": "number"},
        "date": {"type": "string"}
    }
})

results = []
for split in splits.splits:
    extracted = client.extract(markdown=split.markdowns[0], schema=schema)
    results.append({
        "type": split.classification,
        "id": split.identifier,
        **extracted.extraction
    })
```

## 4. Parse Jobs (Async, Large Files)

> See [Parse Jobs Specification](ADE_API.md#4-parse-jobs-api-async) for parameters and response structure.

```python
import time

# Create job for large file (>50MB)
job = client.parse_jobs.create(document=Path("large.pdf"), model="dpt-2-latest")
print(f"Job ID: {job.job_id}")

# Poll for completion
while True:
    status = client.parse_jobs.get(job.job_id)
    print(f"Status: {status.status}, Progress: {status.progress * 100:.0f}%")

    if status.status == "completed":
        result = status.result
        break
    elif status.status == "failed":
        raise RuntimeError(f"Job failed: {status.failure_reason}")

    time.sleep(5)

# List jobs
jobs = client.parse_jobs.list(status="processing")
for j in jobs.jobs:
    print(f"{j.job_id}: {j.status} ({j.progress * 100:.0f}%)")
```

## Error Handling

```python
from landingai_ade.exceptions import (
    RateLimitError,
    APITimeoutError,
    APIStatusError,
    APIConnectionError,
)
import time

def parse_with_retry(client, file_path, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.parse(document=Path(file_path))
        except RateLimitError:
            wait = 2 ** attempt * 10
            print(f"Rate limited, waiting {wait}s...")
            time.sleep(wait)
        except APITimeoutError:
            print("Timeout — switching to parse jobs")
            job = client.parse_jobs.create(document=Path(file_path))
            return poll_job(client, job.job_id)
        except APIConnectionError:
            print(f"Network error, attempt {attempt + 1}/{max_retries}")
            time.sleep(2)
        except APIStatusError as e:
            if e.status_code == 413:
                print("File too large — use parse jobs")
                job = client.parse_jobs.create(document=Path(file_path))
                return poll_job(client, job.job_id)
            raise
    raise RuntimeError("Failed after retries")

def poll_job(client, job_id, timeout=300):
    import time as t
    start = t.time()
    while t.time() - start < timeout:
        status = client.parse_jobs.get(job_id)
        if status.status == "completed":
            return status.result
        if status.status == "failed":
            raise RuntimeError(f"Job failed: {status.failure_reason}")
        t.sleep(5)
    raise TimeoutError("Job did not complete in time")
```

## Async / Concurrent Processing

```python
import asyncio
from landingai_ade import AsyncLandingAIADE
from pathlib import Path

async def parse_multiple(files: list[str]):
    client = AsyncLandingAIADE()

    tasks = [client.parse(document=Path(f)) for f in files]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    for file, result in zip(files, results):
        if isinstance(result, Exception):
            print(f"Failed {file}: {result}")
        else:
            print(f"Parsed {file}: {result.metadata.page_count} pages")

    return [r for r in results if not isinstance(r, Exception)]
```
