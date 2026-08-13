# WF8 — Document → PDF → Drive

Watches Receipts/Invoices/TaxInvoices for records that passed validation, renders each as RTL Hebrew HTML, converts to PDF, uploads to Drive, and writes the PDF link back.

```mermaid
flowchart LR
    classDef trigger fill:#e6f9ee,stroke:#22c55e,stroke-width:2px,color:#111827;
    classDef app fill:#eaf2ff,stroke:#3b82f6,stroke-width:2px,color:#111827;
    classDef logic fill:#fff8e1,stroke:#f59e0b,stroke-width:2px,color:#111827;

    T1["Airtable Trigger<br/><small>Receipts pending</small>"]:::trigger
    T2["Airtable Trigger<br/><small>Invoices pending</small>"]:::trigger
    T3["Airtable Trigger<br/><small>TaxInvoices pending</small>"]:::trigger
    S1["Search Pending"]:::app
    S2["Search Pending"]:::app
    S3["Search Pending"]:::app
    M["Merge Documents"]:::logic
    H["Build HTML<br/><small>RTL Hebrew</small>"]:::logic
    P["Convert to PDF<br/><small>PDFShift</small>"]:::app
    D["Upload to Drive"]:::app
    R["Route by Doc Type"]:::logic
    U1["Update Receipt<br/><small>PdfUrl</small>"]:::app
    U2["Update Invoice<br/><small>PdfUrl</small>"]:::app
    U3["Update TaxInvoice<br/><small>PdfUrl</small>"]:::app

    T1 --> S1 --> M
    T2 --> S2 --> M
    T3 --> S3 --> M
    M --> H --> P --> D --> R
    R --> U1
    R --> U2
    R --> U3
```
