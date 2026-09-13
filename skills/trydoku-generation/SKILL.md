---
name: trydoku-generation
description: Generate document batches through the TRYDOKU REST API using Word (.docx) templates and JSON, CSV, or Excel data. Use for TRYDOKU template preparation, data mapping, batch submission, status checks, downloads, and API integration; not for unrelated Word editing.
---

# TRYDOKU document generation

Generate documents from a local `.docx` template or an existing TRYDOKU template and a dataset with one record per document. Convert CSV or Excel inputs into JSON before calling the API.

## API contract

Verified against the published documentation on **2026-09-13**. Use the [interactive reference](https://www.trydoku.com/docs/api/reference) and [OpenAPI JSON](https://www.trydoku.com/docs/api/api.json) for endpoint details, and the [API overview](https://www.trydoku.com/docs/api) for request examples. Recheck these sources when implementing or changing an integration.

The current generated schema has a conflicting `data.items` definition: `GenerateDocumentRequest` declares strings, while the endpoint describes row objects and the quickstart uses them. Use object rows as shown below; do not stringify each row to satisfy that conflicting definition. The schema also does not enumerate all batch or setup statuses.

Base URL: `https://www.trydoku.com/api/v1`. Paths below are relative to this URL; do not append `/api/v1` twice.

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/generate` | Submit a generation batch. |
| `GET` | `/batches/{batchId}` | Read batch progress and available item results. |
| `GET` | `/batches/{batchId}/zip` | Download the generated archive when ready. |

Read `TRYDOKU_API_TOKEN` from the execution environment and send `Authorization: Bearer <token>` with API requests. Use `Accept: application/json` for JSON responses and `Content-Type: application/json` for submission. Use `Accept: application/zip` for the archive. Check token presence without printing it; local preparation does not require authentication.

### Request shape

Choose one template source:

- `template_uuid`: the UUID of an existing template owned by the authenticated user.
- `template_base64`: standard padded base64 of the bytes of a non-macro `.docx` file, without a data-URL prefix. For example, use Python's `base64.b64encode(template_bytes).decode("ascii")` or Node.js's `templateBuffer.toString("base64")`.

Example using an existing template; replace the illustrative UUID and records with the user's template and data:

```json
{
  "template_uuid": "64dd0a30-187e-486b-b270-a2132b3ec456",
  "data": [
    {
      "client_name": "Example Company",
      "invoice_id": "INV-001",
      "invoice_date": "2026-09-13",
      "amount": 1200,
      "is_discounted": false,
      "items": [
        { "description": "Consulting", "price": 800 },
        { "description": "Implementation", "price": 400 }
      ]
    }
  ],
  "format": "zip"
}
```

For a local template, replace `template_uuid` with `template_base64` and its encoded contents. Serialize the complete payload with a JSON library; do not interpolate unescaped data into shell commands. `format` may be omitted; the published request schema allows `"zip"` or `null`, not `"pdf"` or `"docx"`.

Prefer object rows with keys matching the template variables. For indexed rows, the endpoint also describes `variables` (ordered column names) and `variable_mapping` (source names to placeholder names); consult the live reference before using this form.

### Published request limits

- `data`: 1–500 rows; empty-only rows are removed, while `0` and `false` count as values.
- Decoded template: at most 10 MiB, in a valid non-macro DOCX package.
- Data: at most 5 MiB of canonical JSON, 50,000 nodes, three nested levels, and 50,000 UTF-8 bytes per string.
- Raw HTTP request body: at most 20 MiB. `Content-Encoding` must be absent or `identity`.
- Indexed-row `variables`: 1–100 unique names, at most 128 UTF-8 bytes each. `variable_mapping`: at most 100 unique targets.

Validate size and row count before submission. Split larger datasets into batches within all limits, preserving a mapping back to source rows. Do not truncate records or fields silently.

## Template and data preparation

Follow the [template guide](https://www.trydoku.com/docs). Preserve these marker forms:

| Construct | Example |
| --- | --- |
| Scalar | `{{ client_name }}` |
| Loop start | `{> items }}` |
| Loop end | `{< items }}` |
| Conditional | `{% if is_discounted %}Discount applies.{% else %}Standard price.{% endif %}` |

For repeating table rows, place the loop start in the first cell and the end in the last cell of the same row. Inside the row, use item fields such as `{{ description }}` and `{{ price }}`. Loop markers deliberately use a single opening brace.

Inspect placeholders before generating. The [Word Template Variable Parser](https://www.trydoku.com/tools/word-template-variable-parser) can help when browser tooling is available. For local inspection, parse DOCX XML and join text runs within paragraphs; Word can split a visible marker across runs. Include relevant tables, headers, and footers instead of searching only raw `word/document.xml` text. If the template is unavailable, prepare the data but state that template inspection remains incomplete.

Map source columns explicitly to the template's variable names, preserving their spelling and case. Keep loop values as arrays of objects and conditional values as JSON booleans. Preserve identifiers with leading zeros. Use numeric values for numeric schema fields and `YYYY-MM-DD` for date fields; respect any required fields, ranges, and select options configured on the template. Do not invent missing business values or assume the web importer's column mapping happens automatically in the API.

## Execution workflow

1. **Prepare the requested inputs.** Identify the template, dataset, output directory, and row mapping. If the request is only to inspect or prepare, complete that work locally. Generation sends the template and data to TRYDOKU and consumes account credits; keep submissions within the user's requested scope.
2. **Submit once.** Send the serialized payload to `/generate` with a finite request timeout. A `201` response creates a batch; a `202` response with `GENERATION_SETUP_PENDING` also returns a batch to track. Save `data.id` immediately. Neither response means the documents are ready.
3. **Poll the existing batch.** Read `/batches/{batchId}`, tracking `status`, `setup_status`, `total_items`, `processed_items`, and `failed_items`. Use a bounded wait, for example every 2–5 seconds for up to five minutes, adjusted to the task. Treat these timings as client defaults, not service guarantees. Continue polling a setup-pending batch instead of submitting it again. Stop on an explicit failure. If a status is unfamiliar, inspect the response and current documentation without assuming success. At the deadline, report the batch ID and last known state so polling can resume later.
4. **Check the outcome.** When `status` is `completed`, inspect `failed_items` and any available `items[].error`, `items[].status`, and `items[].row_index`. Do not describe a batch with failed rows as fully successful. Report counts under the API's labels; the reference does not define whether `processed_items` includes failures. Preserve the API row index and the source-row mapping; do not assume the API index is an Excel row number.
5. **Download and verify.** When `data.links.zip` is non-null, retrieve the archive. Before attaching the bearer token to a response-provided URL, verify that its origin is exactly `https://www.trydoku.com`; do not forward credentials to another host or follow redirects blindly. If the link has a different origin, use the documented `/batches/{batchId}/zip` endpoint on TRYDOKU and inspect any redirect separately. Check the HTTP status, save to a temporary file, and verify that it is a readable ZIP before moving it to the requested destination. Do not overwrite unrelated files. Inspect representative generated documents for unresolved markers, incorrect values, and layout problems before reporting the result; state any inspection limits.
6. **Report results.** Provide the batch ID, returned counts, any row failures, and the local archive path. If generation or download failed, describe the actual state and the next recoverable action.

## Errors and retries

| Response | Action |
| --- | --- |
| `401` | Check token availability and validity without exposing it. Password changes or resets revoke existing tokens. |
| `403` | Check access to the template and account restrictions. |
| `402` | Report insufficient credits and the returned `credits_available` / `credits_required` values. |
| `413` | Reduce the request size within the documented limits. |
| `415` | Remove unsupported content encoding. |
| `422` | Use field-level `errors` to correct the payload or template schema mismatch. |
| `503` with `GENERATION_SETUP_FAILED` | Report setup failure; this response states that credits were refunded. Do not assume every `503` includes a refund. |
| ZIP `400` with `BATCH_NOT_READY` | Resume bounded status polling instead of submitting another batch. |

For transient GET failures or HTTP `429`, honor `Retry-After` when provided, otherwise use capped exponential backoff with jitter and an overall deadline. These are client recovery practices; the reference does not specify a fixed rate quota.

Do not automatically replay `POST /generate` after a timeout, connection loss, or ambiguous server response. The published contract does not document an idempotency key, and replaying a request can create another batch and consume credits. If an ID was received, inspect that batch. Otherwise, reconcile the outcome in the account before deciding whether a new submission is appropriate. Retry only failed records when their outcome is known and a retry is within scope.

Keep tokens, encoded templates, and sensitive row values out of logs and version control. Environment variables are not encrypted storage. Download results promptly and consult the current [security documentation](https://www.trydoku.com/security) for data handling; do not infer exact retention or database behavior from this skill. Check [pricing](https://www.trydoku.com/pricing) for current credit terms rather than promising a fixed charge or refund policy.
