# WF13 — App Gateway (chat + writes)

Single webhook endpoint for the admin app. Every request needs a shared-secret header, checked before anything else runs; requests without it never reach the agent or Airtable.

```mermaid
flowchart LR
    classDef trigger fill:#e6f9ee,stroke:#22c55e,stroke-width:2px,color:#111827;
    classDef logic fill:#fff8e1,stroke:#f59e0b,stroke-width:2px,color:#111827;
    classDef agent fill:#f3e8ff,stroke:#7c3aed,stroke-width:2px,color:#111827;
    classDef subnode fill:#f5f0ff,stroke:#a78bfa,stroke-width:2px,color:#111827;
    classDef send fill:#e6fbf8,stroke:#14b8a6,stroke-width:2px,color:#111827;

    W["App Webhook"]:::trigger
    I{"IF<br/><small>Check Secret</small>"}:::logic
    U["Respond<br/><small>Unauthorized (401)</small>"]:::send
    AG["AI Agent<br/><small>App Agent</small>"]:::agent
    CM["OpenAI Chat Model"]:::subnode
    T["7 Airtable tools<br/><small>search + create</small>"]:::subnode
    P["Official policy + Product catalog<br/><small>Qdrant RAG tools</small>"]:::subnode
    R["Respond"]:::send

    W --> I
    I -- false --> U
    I -- true --> AG --> R
    AG -.-> CM
    AG -.-> T
    AG -.-> P
```
