# Fault-Tolerant Enterprise AI Lead Orchestrator
⚡ **An Production-Grade Event-Driven Automation Engine by F22mation**

This repository contains the blueprints and architecture for a highly resilient, self-hosted, and fault-tolerant Lead Management and AI Qualification System built on **n8n (Docker-based)**. Designed with a strict "Design for Failure" philosophy, this infrastructure safely ingests, deduplicates, evaluates, tracks, and asynchronously routes multi-channel leads without data loss, even during infrastructure downtime, network drops, or API rate-limiting.

---

## 🗺️ System Architecture

Below is the end-to-end data flow showing how lead ingestion, idempotency hashing, asynchronous AI evaluation, and state-enforced routing interact inside the core architecture:

```mermaid
graph TD
    A[Webhook Ingestion] --> C[Set & Sanitize Node]
    B[Telegram Bot Ingestion] --> D[Set & Sanitize Node]
    B --> B1[Send UI Message: Please Wait...]
    
    C --> E[Idempotency Engine: Unique Hash Code]
    D --> E
    
    E --> F{AI Agent Evaluation}
    F -- Success --> G[Extract Lead Score 0-100]
    F -- Failure / Timeout --> H[Set Score = -1 & Flag: 'AI Failed']
    
    G --> I[Merge Node]
    H --> I
    
    I --> J[JavaScript Priority Cleansing Node]
    J --> K[Fetch Central Sheet State]
    K --> L[JS IsNew? Evaluator]
    
    L -- New Lead [True] --> M[Append to Google Sheets]
    L -- Existing / Retried Lead [False] --> N{VIP Logic Router}
    
    M --> N
    
    N -- VIP [Score > 80 OR Budget > $10k] --> O[Get Row State]
    O --> P{Is telegram_notified Empty?}
    P -- Yes --> Q[Rate-Limit Loop & Wait Queue] --> R[Send Telegram Message + Inline Buttons] --> S[Update Sheet Flag]
    P -- No --> T[Await Admin Decision]
    
    N -- Non-VIP [False] --> U[Get Row State]
    U --> V{Is email_sent Empty?}
    V -- Yes --> W[Channel Source Switch] --> X[Send Autoreply Email] --> Y[Update Sheet Flag]
    
    %% Callback Flow
    Z[Admin Clicks Approve/Reject Button] --> AA[Tunnel Interface] --> AB[Process Callback Node]
    AB --> AC[Update Status Formally in Sheet] --> AD[Wipe Inline Buttons + Update Message Text]
    AD -- Approved --> AE[Trigger Welcome Onboarding Email]
