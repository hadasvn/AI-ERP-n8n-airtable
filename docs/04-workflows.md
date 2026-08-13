# Building the workflows

## Credentials (n8n → Credentials → New)

| Credential | Used by |
|---|---|
| Airtable Personal Access Token | most workflows |
| Telegram — manager bot | WF9 |
| Telegram — customer service bot | WF5 |
| Gmail (OAuth2) | WF4a, WF4b |
| Google Drive (OAuth2) | WF8 |
| OpenAI — chat | WF5, WF9, WF13, and the reply-drafting step in WF4b |
| OpenAI — embeddings (multilingual) | WF6, WF7, and the RAG query tools in WF5/WF9 |
| Qdrant Cloud | WF6, WF7, and the RAG query tools in WF5/WF9 |

## WF1a / WF1b / WF1c — Tax-document validation (Receipts / Invoices / TaxInvoices)

One workflow per table instead of a single shared workflow with a switch — simpler to build and debug independently. Each: **Airtable Trigger** (on the table's `Created`/`Last modified` field) → **Search** (all records, to find the current max `DocNumber`) → **Code** (assign the next sequential number; for Invoices/TaxInvoices also validate the VAT rate — 18% from 2025-01-01, 17% before) → **Update** (write back only `DocNumber`, `Status`, `ValidationError`).

Idempotent by design: a record that already has a `DocNumber` doesn't get renumbered.

⚠️ **Real bug hit here**: the Update node's "Values to Update" was initially built with *every* field on the table mapped, but only `DocNumber`/`Status`/`ValidationError` had a real expression — every other field (`Subtotal`, `Total`, `Amount`, `CustomerName`, dates…) was blank or a literal `0`, so each run silently zeroed out real data. Fix: the Update node should only list the fields it's actually meant to touch — remove the rest rather than leaving them mapped to nothing.

## WF3 — Contact intake + dedupe

**Webhook** → **Search** (Airtable, `Filter By Formula: {Email} = "{{ $json.body.Email }}"`, with *Always Output Data* on) → **IF** (`id` **is not empty** → duplicate) → true branch: no-op; false branch: **Create a record**.

⚠️ Watch two easy mistakes here: the IF operator needs to be "is not empty" (not "is empty"), and the true/false branches need to connect to the right side of the Create node — both were initially backwards, so a detected duplicate still created a new record.

## WF4a — Sales agent, cold emails

Scheduled trigger → search Leads with `Status = New` → chat model drafts the email → **Gmail: Send** → update the lead (`Status = Contacted`, `LastContactedAt`).

## WF4b — Sales agent, reply check

**Gmail Trigger** → match the incoming email to a Lead by thread/email → **IF** (is this lead's status still `Contacted`, i.e. a real reply worth handling) → pull the **full Gmail thread** (not just the latest message) → AI agent (with the same policy RAG tool WF5/WF9 use) drafts a reply → mark the lead `Replied`.

## WF5 — Customer service agent

**Telegram Trigger** (customer-service bot) → AI agent with two RAG tools (policy search, product search — both Qdrant *Retrieve-as-Tool*) → **Telegram: Send**. Grounded entirely in the two Qdrant collections; the system prompt explicitly tells the model to say it doesn't know rather than invent an answer.

## WF6 / WF7 — Embedding pipelines (Qdrant)

Same shape for both: trigger → **Default Data Loader** → **Character Text Splitter** (policies only — the product CSV rows are already short) → **Qdrant Vector Store** (mode: Insert). Two separate Qdrant collections (`ai_erp_policies`, `ai_erp_products`) so the two domains never mix in retrieval.

Run both by hand after any n8n instance restart or credential change — the alternative to Qdrant here is n8n's built-in in-memory vector store, which is wiped on every restart; that's the reason this build uses Qdrant instead.

## WF8 — Document → PDF → Drive

Three Airtable triggers (Receipts/Invoices/TaxInvoices, on `Status = Pending`) → **Merge** → **Build HTML** (RTL Hebrew, from `templates/invoice-template.html`) → **Convert to PDF** → **Upload to Drive** → **Switch** (route by doc type) → three **Update** nodes writing `PdfUrl` back to the source record.

The Airtable Trigger's polling field has to be a real Created/Last-modified time field — Receipts and TaxInvoices needed one added manually (see `docs/01-airtable.md`), since only Invoices had one from the start.

⚠️ **Real bug hit here**: a record missing its document ID stayed in `Pending` permanently, so the once-a-minute trigger kept re-processing it — Build HTML classified it as `docType: "Unknown"` every time, matched no branch in the Switch, and silently wrote a fresh empty PDF to Drive every minute. Fix: give the record a valid ID, and treat "stuck records creating junk files" as a routine thing to check for in a polling pipeline like this.

## WF9 — Manager agent

**Telegram Trigger** (manager bot) → owner check → AI agent → **Telegram: Send**.

Tools: Search on all 4 core tables (Invoices, Leads, Products, Tasks — read-only), the two Qdrant RAG tools, and three billing-document creation tools (`create_invoice`, `create_tax_invoice`, `create_receipt`) added after the initial build.

⚠️ **Real bugs hit while adding the billing tools**:
- `VATRate` is a **decimal fraction** (`0.18` for 18%), not a whole number — sending `18` produces a VAT amount 100x too large. Both the system prompt and each tool's description need to say this explicitly, or the model defaults to the "natural" whole-number reading.
- Leaving `DocNumber` mapped through the model (so it *could* stay empty, since numbering is WF1's job) doesn't work as expected: an empty AI-provided value coerces to `0` in a JS expression (`Number('') === 0`), not to a blank field. The fix is to remove the `DocNumber` mapping from the creation tools entirely rather than trying to pass it through empty.
- A generic-sounding tool name/description (or a typo in one) is enough to make the model call the wrong tool for a borderline question — tool names and descriptions are effectively part of the prompt, not just internal labels.

## WF13 — App Gateway (chat + writes)

**Webhook** → check a shared-secret header (`x-app-secret`) against an n8n-side value — requests without a valid header get **401** and never reach the agent or Airtable → single AI agent (same tool set as WF9's billing tools, plus task creation) → **Respond to Webhook**.

This is what the Lovable admin app's chat panel calls. The shared secret must be attached **server-side** (an edge function/serverless proxy), never embedded in client-side code — a secret embedded in the browser bundle is visible in devtools regardless of any header check on the n8n side.

Record creation from the admin app's table screens (Invoices/TaxInvoices/Receipts/Tasks) writes directly to Airtable from the app instead of going through this agent — routing a form with fully-known fields through an LLM added latency and let the model get asked about internal fields (like `DocNumber`) that should never be user-facing. The chat panel itself still goes through WF13.
