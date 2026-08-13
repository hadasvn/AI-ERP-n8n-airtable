# Telegram bots

Two separate bots, created via [@BotFather](https://t.me/BotFather), each with its own token in its own n8n Telegram credential:

| Bot | Role | Used by |
|---|---|---|
| Manager bot ("AI Electronics - מנהל") | Owner-only: business Q&A, task/billing-document creation | Workflow 9 |
| Customer service bot ("customer service yossi") | Public-facing: product + policy questions, RAG-grounded | Workflow 5 |

Each bot needs its own **Telegram Trigger** node in its workflow. When copying a Telegram Trigger node between workflows in n8n, delete and re-add it rather than duplicating it — a duplicated trigger keeps the source's webhook path and will conflict, blocking activation.
