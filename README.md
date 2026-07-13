
# AI Lead Automation System (F22mation)

A production-grade, self-healing Lead Management System built with n8n.

## System Architecture

```mermaid
flowchart LR
    A[Webhook / Telegram] --> B[Merge & Normalize]
    B --> C[AI Scoring (OpenAI)]
    C --> D[Priority & Cleanup]
    D --> E[Dedup Check (Sheets API)]
    E --> F{New Lead?}
    F -->|Yes| G[Append Row]
    F -->|No| H[Get Existing Row]
    G --> I[Route by VIP]
    I --> J[VIP: Telegram to Admin]
    I --> K[Non-VIP: Email / Telegram]
    J --> L[Admin Callback]
    L --> M[Update Status & Notify]
    K --> N[Flag Updates]
    M --> N
    H --> I
    O[Scheduler (Sweeper)] --> P[Recover Stuck Leads]
    P --> I
```

Key Features

· Omnichannel Input – Webhook (Site forms) and Telegram bot.
· Deterministic Idempotency – Custom hash-based unique_id + Google Sheets API direct lookup prevents duplicate processing.
· AI Lead Scoring – OpenAI integration with fault-tolerant fallback (lead_score = -1 on failure).
· VIP Routing – High-budget (> 10000) or high-score (> 80) leads are sent to admin via Telegram with inline approval keyboard.
· Stateful Flagging – Three notification flags (email_sent, telegram_notified, welcome_email_sent) guarantee exactly-once delivery.
· Self-Healing Sweeper – Scheduled workflow that rescues leads stuck in pending state or missed notifications due to downtime/crashes. Respects race conditions via created_at timestamp (10‑minute delay). Re-sends pending VIP notifications after timeout.
· Human-in-the-Loop – Inline keyboard approval, automatic button removal after decision, edited message confirmation.
· Source-Aware Routing – Leads from Telegram receive Telegram confirmations; Webhook leads get emails. Uses source field and Switch node.
· Error Watchdog – Separate Error Trigger workflow alerts admin on critical failures.

Technologies & APIs

· n8n (Self-hosted / Docker)
· Google Sheets API (REST, OAuth2)
· Telegram Bot API (Webhooks, Inline Keyboards, Message Editing)
· OpenAI API (Chat Completions)
· Gmail API (Sending emails)
· Serveo (Public HTTPS tunnel for local development)

Installation

1. Import the workflow JSON files from this repo into n8n.
2. Set up credentials:
   · Google OAuth2 (Sheets, Gmail)
   · Telegram Bot Token
   · OpenAI API Key
3. Expose your local n8n webhook URL using a tunnel (e.g., Serveo) and update the WEBHOOK_URL environment variable.
4. Activate the main workflow, the Sweeper, and the Error Watcher.
5. (Optional) Adjust Google Sheets columns as described in schema.md.

Screenshots & Demo

(Add your own screenshots and a short video link later)

License

MIT
