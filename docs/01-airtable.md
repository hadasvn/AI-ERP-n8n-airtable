# Airtable setup

## Base

Base name: **AI-ERP**. The base ID appears in the URL after `airtable.com/` and starts with `app`.

## Tables

Build the 8 tables from [`schema/schema.json`](../schema/schema.json): Customers, Leads, Products, Invoices, TaxInvoices, Receipts, Tasks, Files.

## Manual field fix (required)

The Airtable API cannot create "Created time" / "Last modified time" fields — they have to be added by hand in the UI (**+** in the field row → choose the field type):

- **Invoices**, **Leads**: add `Created`, type **Created time**.
- **TaxInvoices**, **Receipts**: add `Created`, type **Last modified time** (not Created time) — this is what lets the PDF-generation trigger (WF8) also fire when a record's `Status` is updated after validation, not only when the record is first created.

Without this field, the corresponding Airtable Trigger node in n8n has nothing to poll on.

## Personal Access Token

Create one at `airtable.com/create/tokens`, scoped to this base, with read + write access to both records and schema (schema access is needed if you build/modify tables programmatically rather than by hand in the UI). Enter it into n8n's Airtable credential.

## Data model notes

- Relationships (`CustomerId`, `RelatedInvoice`, …) are plain-text identifiers, not Airtable's native link field.
- `VATRate` is a **percent** field storing a decimal fraction — `0.18` for 18%, not `18`. Sending a whole number here silently produces a 100x-inflated VAT amount; this is a real bug that surfaced during the build (see `docs/04-workflows.md`, workflow 13).
