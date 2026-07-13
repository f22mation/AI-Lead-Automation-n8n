<!-- 1. HERO IMAGE -->
![F22mation Lead Engine Banner](https://github.com/f22mation/AI-Lead-Automation-n8n/raw/main/assets/hero-banner.png) 
*Note: If the banner image above does not load, please check the assets folder.*

<!-- 2. TITLE -->
# AI-Powered Lead Lifecycle Engine

<!-- 3. SHORT SUMMARY -->
A production-ready, self-healing enterprise automation system built on self-hosted n8n (Docker). This engine orchestrates the entire lifecycle of a business lead—handling multi-channel ingestion, intelligent AI qualification, real-time human-in-the-loop validation, and asynchronous follow-up loops. 

Unlike fragile, basic automations, this system is architected to prioritize data integrity, survive API downtimes, and actively recover stalled operations without missing a single revenue opportunity.

---

<!-- 4. WHY THIS PROJECT? (BUSINESS VALUE) -->
## 🎯 Why This System Matters (Business Value)

In high-volume sales operations, traditional automations fail quietly when networks blink or APIs hit rate limits. This system solves critical business leakage:
* **Zero Lost Leads:** Bridges the gap between third-party webhooks and communication channels with robust fault-tolerance frameworks.
* **API Failure Survival:** If OpenAI or your database drops, the system seamlessly logs the error state and processes the lead instead of crashing mid-way.
* **Spam & Duplicate Protection:** Enforces strict state tracking to prevent sending duplicate notifications or charging your cloud/AI accounts for identical entries.
* **Automated Data Recovery:** Features an asynchronous "Sweeper" routine that sweeps the database every few minutes to find, fix, and re-route leads stuck in limbo.

---

<!-- 5. FEATURES -->
## ⚡ Core Features

* **✅ Omnichannel Ingestion:** Unified pipeline for both RESTful Webhooks and interactive Telegram bots.
* **✅ AI Lead Scoring:** Dynamically evaluates, scores, and sanitizes incoming lead data using OpenAI GPT models.
* **✅ Idempotency & Anti-Spam Gates:** Validates system status flags before any outbound message or email to block duplicate notifications.
* **✅ Human-in-the-Loop Interactivity:** Interactive Telegram inline buttons allowing admins to approve or reject leads directly from their chats.
* **✅ Concurrent State Locking:** Disables Telegram interface buttons instantly upon click to completely prevent race conditions.
* **✅ Rate-Limit Mitigation:** Integrated intelligent delay loops ensuring high-volume execution safely clears Telegram API constraints.
* **✅ Asynchronous Sweeper (Reconciliation):** An automated cron-based background system that captures and revives any pending leads older than 10 minutes.
* **✅ Fault-Tolerant Error Catching:** Isolated fallback nodes protecting the main runtime pipeline from structural failure during API drops.

---

<!-- 6. DEMO VIDEO & GIFS -->
## 📺 Live Demo & Walkthrough

### System Operation Demo (2-Minute Video)
[![Watch the Demo Video](https://img.shields.io/badge/YouTube-Watch%20Video-red?style=for-the-badge&logo=youtube)]([لینک ویدیو یوتیوب یا آپارات خود را اینجا قرار دهید])

### Desktop & Mobile Previews
| Interactive Telegram Bot Approval | Google Sheets Synchronization |
| :---: | :---: |
| ![Telegram Demo](https://github.com/f22mation/AI-Lead-Automation-n8n/raw/main/assets/telegram-demo.gif) | ![Google Sheets Demo](https://github.com/f22mation/AI-Lead-Automation-n8n/raw/main/assets/sheets-demo.gif) |

---

<!-- 7. ARCHITECTURE DIAGRAM -->
## 🗺️ System Architecture

This Mermaid diagram illustrates the automated data flow, state validation gates, and the background recovery loops:

```mermaid
graph TD
    %% Ingestion
    A[Webhook Ingestion] -->|POST Payload| C[Normalization Gate]
    B[Telegram Interface] -->|User Message| C
    B -->|Acknowledge| B1[Instant UI Receipt Receipt]
    
    %% Processing & AI
    C -->|Unified Stream| D[AI Scoring Agent]
    D -->|Success| E[Priority Evaluation]
    D -->|API Failure Fallback| D1[Error Logger Node]
    D1 --> E
    
    %% Validation & DB
    E --> F[Deduplication Check]
    F -->|Unique Lead| G[Database Insertion]
    F -->|Duplicate Lead| F1[Discard Stream]
    G -->|Database Error Fallback| G1[Admin Crisis Notification]
    
    %% Routing Gates
    G --> H{Priority Check}
    H -->|VIP Lead| I[Idempotency Status Check]
    H -->|Non-VIP Lead| J[State Flag Check]
    
    %% Sweeper Loop (Asynchronous)
    K[Scheduled Sweeper Cron] -->|Fetch Pending > 10m| L[Time Logic Check]
    L -->|Stuck State Detected| M[Re-Route Back to Admin Chat]
