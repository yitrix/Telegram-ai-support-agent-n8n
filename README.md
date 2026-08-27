# AI Customer Support & Lead-Qualification Bot (n8n + RAG + Postgres)

A self-hosted, AI-powered customer support agent that answers customer questions from a company's Returns & Refunds policy, automatically qualifies and saves leads, and escalates unresolved queries to a human via email — built entirely on open-source, self-hosted infrastructure.

## What it does

- Customers message a Telegram bot with questions (e.g. "when will I get my refund?")
- An AI agent answers using **Retrieval-Augmented Generation (RAG)** over a company policy document — never guessing or hallucinating answers
- If the customer shows buying/support intent, the agent captures their name, contact, and intent, and saves it as a lead
- If a question falls outside the knowledge base, or the customer asks for a human, the workflow automatically escalates: updates the lead status and sends a real-time email alert
- Every conversation turn is logged for auditability and future context

## Architecture

```
Telegram Trigger
      │
      ▼
   AI Agent ──── Tool: RAG search (Postgres + pgvector, Returns/Refunds policy)
      │      └── Tool: save_lead (Postgres upsert)
      │      └── Memory: per-chat conversation memory (Postgres)
      ▼
Structured Output Parser (JSON: reply, lead_captured, name, contact, intent, needs_human, escalation_reason)
      │
      ▼
     IF (needs_human?)
   ┌──true──────────────┐   false
   ▼                     │     │
Update lead → escalated  │     │
Gmail alert to team      │     │
   └─────────┬───────────┘     │
             ▼                 ▼
        Log conversation → Telegram reply to customer
```

**Separate ingestion pipeline** (run once, or whenever the policy doc changes):
```
Read file from disk → Extract text (PDF) → Chunk/load → Generate embeddings → Store in Postgres (pgvector)
```

## Tech stack

- **n8n** — self-hosted (Community Edition), orchestration engine
- **PostgreSQL + pgvector** — vector store for RAG, plus relational storage for leads and conversation logs
- **Mistral Cloud** — LLM (chat + embeddings)
- **Telegram Bot API** — customer-facing channel
- **Gmail API (OAuth2)** — human escalation alerts
- **Docker Compose** — full local/self-hosted deployment
- **ngrok** — public HTTPS tunnel for Telegram webhook delivery during development

## Why self-hosted

This project intentionally avoids paid SaaS automation tiers. Every component — the workflow engine, vector database, and LLM provider — runs on free-tier or self-hosted infrastructure, making the same architecture usable by a small business without ongoing platform fees.

## Setup

1. Clone this repo and copy the environment template:
   ```bash
   cp .env.example .env
   ```
2. Fill in your own values in `.env` (Postgres credentials, ngrok domain/token, sandbox API keys — see `.env.example` for the full list).
3. Start the stack:
   ```bash
   docker compose up -d
   ```
4. Import `workflow.json` into n8n (`localhost:5678` → Workflows → Import from File).
5. Connect your own credentials inside n8n for: Telegram Bot API, Gmail OAuth2, Mistral Cloud, and Postgres.
6. Run the ingestion workflow once to embed your own policy document into the vector store.
7. Message your Telegram bot to test.

## Known limitations

- Single-channel (Telegram) — WhatsApp Cloud API integration planned next
- Single-document knowledge base (Returns & Refunds policy) — designed to extend to multi-document RAG
- Escalation channel is email only; Slack/SMS alerting could be added

## Notes for reviewers

This project was built to demonstrate practical, production-style automation: multi-tool agentic AI, structured output validation, conditional business logic, and persistent state — not just a simple chatbot demo.
