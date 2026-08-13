# AGENTS.md — AI-ERP (n8n + Airtable)

Compact guide for AI agents working in this repo. `README.md` has the full write-up; read it for anything not covered here.

## Project

A small ERP for a fictional Israeli electronics business, built with **n8n**, **AI agents**, and **RAG**. Scope is n8n Cloud + Airtable only — no Docker, no backend, no local app code lives here. Airtable is the database and the UI for raw data; n8n Cloud runs the automation; documents render to PDF and live in Google Drive; a separate Lovable-built admin app talks to n8n over a secured webhook (see `docs/04-workflows.md`, workflow 13).

**The live n8n instance is the only source of truth for workflows.** The repo keeps no JSON exports — `docs/screenshots/` holds canvas and execution screenshots for reference only.

## Commands

None. There is nothing to build, install, run, test or lint here — the repo holds the schema, the RAG content, the document template, and the docs. All execution happens in n8n Cloud.

`schema/schema.json` is the source of truth for the 8 tables. Build them in the Airtable base by hand and add records yourself; six of them need a manually-added `Created`/`Last modified` time field (the Airtable API can't create that field type — see `schema/schema.json` → `notes`).

## Architecture

```
                     ┌──────────────┐
  Telegram bot #1 ──▶│              │──▶ Airtable   (the database)
  (manager)          │              │
  Telegram bot #2 ──▶│  n8n Cloud   │──▶ Gmail      (sales emails)
  (customers)        │  + AI agents │──▶ Drive      (PDF documents)
                     │  + Qdrant    │
  Gmail  ───────────▶│              │◀── Lovable admin app (webhook,
  Schedules ────────▶└──────────────┘     shared-secret header)
```

n8n Cloud provides the public HTTPS URL, so Telegram triggers and webhooks work with no tunnel.

### Models

Set in n8n credentials, swappable — no workflow is tied to a vendor.

- **Chat**: any OpenAI-compatible endpoint.
- **Embeddings**: a separate OpenAI-compatible endpoint, multilingual (Hebrew content).

## Repo layout

```
schema/schema.json     ← single source of truth (8 tables). Everything reads this.
mock/policies/         ← policy/business-rule docs (*.md) for RAG — embedded into Qdrant
mock/products/         ← product catalog (CSV) — embedded into Qdrant, also loaded into the Products table
templates/              ← invoice/receipt/tax-invoice HTML (RTL Hebrew), rendered to PDF
docs/                  ← setup notes + workflow canvas/execution screenshots
```

## Deviations from the course reference build

- **RAG store is Qdrant Cloud, not n8n's in-memory Simple Vector Store** — avoids the store being wiped on every n8n instance restart. Two collections: `ai_erp_policies`, `ai_erp_products`.
- **Workflow 1 (tax-doc validation) is split into three workflows** (Receipts / Invoices / TaxInvoices) instead of one, for simplicity.
- **Workflow 13 (App Gateway)** is an addition beyond the official 9 workflows: a single webhook-triggered AI agent that powers a separate Lovable admin dashboard (chat + record writes), protected by a shared-secret header checked server-side.
- **The manager agent (workflow 9) has real read tools** (Airtable search on all 4 core tables + the policy/product RAG) instead of only pre-computed summarized data.

## Known limitations (real, hit during this build)

- **Airtable triggers poll at ≥1 minute**, and fire off a `Created`/`Last modified` time field (Airtable has no true "on create" event).
- **Sequential invoice numbering can race** if two documents are created inside the same poll window.
- **Relationships are string foreign keys** (e.g. `CustomerId`), not Airtable links.
- **`VATRate` is stored as a decimal fraction** (`0.18`, not `18`) — a real bug surfaced from this exact ambiguity during the build.

## Conventions

- All user-facing text and document content is **Hebrew (RTL)**.
- `schema/schema.json` is the single source of truth — everything reads it.
- Secrets live in n8n credentials only — never in source, never in git.
