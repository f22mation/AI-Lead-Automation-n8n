# AI Lead Automation System – Full Stack Workflow (F22mation)

A self-healing, event-driven lead management engine built entirely in n8n. The system ingests leads from **Webhook** and **Telegram**, scores them with **OpenAI**, enforces **deterministic deduplication**, routes **VIP leads** for admin approval via interactive Telegram buttons, and guarantees zero missed notifications through a **time‑aware Sweeper**. All processing is idempotent, race‑condition resistant, and designed for production environments that must survive network outages and API failures.

---

## Table of Contents
- [System Architecture](#system-architecture)
- [Core Engineering Principles](#core-engineering-principles)
- [Detailed Component Breakdown](#detailed-component-breakdown)
  - [1. Ingestion & AI Processing Pipeline](#1-ingestion--ai-processing-pipeline)
  - [2. Dynamic Routing & Idempotency Gates](#2-dynamic-routing--idempotency-gates)
  - [3. Admin Interactivity & UI State Lock](#3-admin-interactivity--ui-state-lock)
  - [4. Asynchronous Sweeper (Lead Recovery Engine)](#4-asynchronous-sweeper-lead-recovery-engine)
- [Stateful Flagging & Exactly-Once Semantics](#stateful-flagging--exactly-once-semantics)
- [Error & Fault Tolerance](#error--fault-tolerance)
- [Technologies & APIs](#technologies--apis)
- [Installation & Quick Start](#installation--quick-start)
- [Workflow Files](#workflow-files)
- [License & Contact](#license--contact)

---

## System Architecture

The project consists of three n8n workflows:
1. **Main Workflow** – Core lead processing, routing, and admin interaction.
2. **Sweeper Workflow** – Scheduled recovery for stuck or un‑notified leads.
3. **Error Watcher Workflow** – Global error alerts via Telegram.

The diagram below illustrates the logical flow of the Main Workflow, including the interplay with the Sweeper and external services.
```
                              +-------------------+
                              |   Webhook (Site)  |
                              +--------+----------+
                                       |
                              +--------v----------+
                              |   Telegram Bot    |
                              +--------+----------+
                                       |
                         +-------------v-------------+
                         |   Merge & Normalization   |
                         +-------------+-------------+
                                       |
                         +-------------v-------------+
                         |   AI Agent (OpenAI)       |
                         |   Fault-tolerant scoring  |
                         +--------+--------+---------+
                                  |        |
                         success  |        | error / fallback
                                  |        v
                                  | +------+------+
                                  | | Ai Failure  |
                                  | | lead_score=-1
                                  | +------+------+
                                  |        |
                         +---------v--------v---------+
                         |      Priority & Cleanup    |
                         +----------------+----------+
                                          |
                         +----------------v----------+
                         |   Dedup Check (Sheets API)|
                         +----------------+----------+
                                          |
                                     +----v----+
                                     | New?    |
                                     +----+----+
                                          |
                              Yes         |          No
                         +-----v-----+    |    +-----v-----+
                         | Append Row|    |    | Get Existing|
                         +-----+-----+    |    +-----+-----+
                               |          |          |
                               +-----+----+-----+----+
                                     |
                         +-----------v-----------+
                         |   IF VIP (budget>10k  |
                         |        or score>80)   |
                         +-----------+-----------+
                                     |
                       True          |          False
                  +------v------+    |    +------v------+
                  |  VIP Path   |    |    | Non-VIP Path|
                  +------+------+    |    +------+------+
                         |           |           |
                +--------v--------+  |  +--------v--------+
                | Get Row + Flag  |  |  | Get Row + Flag  |
                | (telegram_notif)|  |  | (email_sent)    |
                +--------+--------+  |  +--------+--------+
                         |           |           |
                    empty?           |      empty?
                    +--v--+          |      +--v--+
                    | Yes |          |      | Yes |
                    +--+--+          |      +--+--+
                       |             |         |
                +------v-------+     |  +------v------+
                | Loop & Wait   |     |  | Switch by   |
                | (rate‑limit)  |     |  | source (Site |
                +------+-------+     |  | / Telegram)  |
                       |             |  +------+------+
                +------v-------+     |         |
                | Send Telegram |     |  +------v------+
                | (Inline       |     |  | Gmail or    |
                | Keyboard)     |     |  | Telegram     |
                +------+-------+     |  +------+------+
                       |             |         |
                +------v-------+     |  +------v------+
                | Update Row    |     |  | Update Row  |
                | (telegram_notif)   |  | (email_sent) |
                +-----------------+  |  +------+------+
                                     |
         +---------------------------+
         |
         |   (Callbacks & Sweeper below)
         |
         v
+-------------------+     +-------------------+
| Telegram Callback |     |  Schedule Trigger |
| Trigger           |     |  (Sweeper)        |
+--------+----------+     +---------+---------+
         |                          |
         |  +-----------+          |
         |  | Answer     |          |
         |  | Callback   |          |
         |  | Query      |          |
         |  +-----+-----+          |
         |        |                |
         |  +-----v-----+          |
         |  | Update Row|          |
         |  | (status)  |          |
         |  +-----+-----+          |
         |        |                |
         |  +-----v-----+         |
         |  | Get Row    |         |
         |  +-----+-----+         |
         |        |                |
         |  +-----v-----+         |
         |  | IF Approved|        |
         |  +-----+-----+         |
         |        |                |
         |    Approved & not sent  |
         |        |                |
         |  +-----v-----+         |
         |  | Switch by  |         |
         |  | source     |         |
         |  +--+-----+--+         |
         |     |     |            |
         |  Site Telegram         |
         |     |     |            |
         | +---v-+ +-v---+        |
         | |Gmail| |Tele.|        |
         | +--+--+ +--+--+        |
         |    |       |           |
         | +--v-------v--+        |
         | | Update Row   |        |
         | | (welcome_sent)|       |
         | +--------------+        |
         |                         |
         v                         v
+-------------------+     +-------------------+
| Edit Message      |     | HTTP Request      |
| (Remove Keyboard) |     | (Read Sheet)      |
+-------------------+     +--------+----------+
                                    |
                          +---------v---------+
                          | Code Node         |
                          | (Filter & Decide) |
                          +---------+---------+
                                    |
                          +---------v---------+
                          | Loop Over Items   |
                          +---------+---------+
                                    |
                          +---------v---------+
                          | Switch (3 outputs)|
                          +--+----+----+-----+
                             |    |    |
                      send_email |    send_welcome
                                 |
                           send_telegram
                             |
                    +--------v--------+
                    | Send Telegram   |
                    | or Gmail        |
                    +--------+--------+
                             |
                    +--------v--------+
                    | Update Row      |
                    | (flag update)   |
                    +-----------------+



```

The **Sweeper** (scheduled every 5 minutes) scans the sheet for rows where `created_at` is older than 10 minutes and notification flags are still `empty` (or VIP leads stuck in `pending`), then re‑injects them into the appropriate notification path.

---

## Core Engineering Principles

- **Deterministic Idempotency:** Every lead carries a content‑based `unique_id` (MD5‑like hash of normalized fields). Before inserting, we query the Google Sheets API directly to check for an existing row, preventing duplicate processing.
- **State Enforcement via Flags:** Three dedicated columns (`email_sent`, `telegram_notified`, `welcome_email_sent`) act as a state machine. Each notification is guarded by a check on the respective flag, guaranteeing **exactly‑once delivery**.
- **Race‑Condition Mitigation:** The Sweeper respects a 10‑minute `created_at` threshold, ensuring it never competes with the main workflow’s freshly inserted leads.
- **Fault‑Tolerant AI:** On AI service failure, the pipeline continues with `lead_score = -1` and `priority = Unknown`. No lead is lost.
- **Human‑in‑the‑Loop UX:** Inline approval buttons in Telegram are physically removed after the admin’s decision to prevent double‑click errors, and the original message is edited to reflect the outcome.

---

## Detailed Component Breakdown

### 1. Ingestion & AI Processing Pipeline
- **Omnichannel Input:** Leads arrive either via a REST Webhook (from a website form) or as direct messages to a Telegram bot. An immediate receipt confirmation is sent back to Telegram users.
- **Data Normalization:** A `Set` node for each channel maps the fields to a common schema (`name`, `email`, `company`, `budget`, `message`, `source`). Telegram leads receive empty `email`/`company` and `budget=0`.
- **Unique ID Generation:** A JavaScript Code Node creates a deterministic hash (`uid-xxxxxxxx`) from the concatenated, normalized string of the lead’s core fields, using a simple 32‑bit hash function (no external crypto required).
- **Merge Node:** The two normalized streams are unified into a single data structure before entering the AI stage.
- **AI Scoring:** An AI Agent node (connected to OpenAI or any compatible model) evaluates the lead’s text and returns a `lead_score` between 0 and 100. The node is configured with **On Error = Continue**, redirecting failures to a dedicated “Ai Failure” Set node that assigns `lead_score = -1` and `Ai_status = "Ai Failed"`.
- **Priority & Cleanup:** A second `Merge` combines success and error outputs. A Code Node then extracts the numeric score (using regex against the raw AI output), determines `priority` (`High` for score>80 or budget>10000, `Normal` otherwise, `Unknown` on failure), and ensures all fields are present and clean.

### 2. Dynamic Routing & Idempotency Gates
- **Dedup Check:** Instead of the built‑in `Get Rows` node (which fails when no row exists), we use an **HTTP Request** directly to the Google Sheets REST API (`/values/{Sheet}!F:F`) to retrieve all existing `unique_id` values. A subsequent Code Node compares the current lead’s `unique_id` against the array.  
  - **New lead:** `is_new = true`, proceed to Append Row.  
  - **Duplicate:** `is_new = false`, skip Append and continue to notification routing.
- **Append Row:** Writes the full lead record into the sheet, including all flags set to `empty` and the `created_at` timestamp (`Date.now()`). This node has **Retry on Fail** enabled (3 attempts, 5s delay) and **On Error = Continue**, sending an alert to the admin on failure.
- **VIP vs. Non‑VIP Routing:** An IF node checks `budget > 10000 OR lead_score > 80`.  
  - **True (VIP):** The flow fetches the row from the sheet, then tests if `telegram_notified` is `empty`. If so, it passes the lead through a **Loop Over Items** (to handle potential batches) with a **Wait** node (to respect Telegram rate limits) before sending a Telegram message to the admin containing the lead details and an **Inline Keyboard** (Approve/Reject) whose callback data includes the `unique_id`. After successful send, the sheet is updated with `telegram_notified = "Sent 👍🏻 " + Date.now()`.
  - **False (Non‑VIP):** The flow reads the row, checks if `email_sent` is `empty`, and then uses a **Switch** node on the `source` field:  
    - `Site` → send a generic confirmation email via Gmail, then update `email_sent` to `"Sent 👍🏻"`.  
    - `Telegram` → send a confirmation message back to the user’s chat (using the stored `telegram_chat_id`), then update `email_sent`.

### 3. Admin Interactivity & UI State Lock
- A separate **Telegram Trigger** (Callback) listens for button presses.
- **Concurrent Handling:** Two parallel branches immediately fire:  
  1. An HTTP Request to `editMessageReplyMarkup` removes the inline keyboard, preventing duplicate clicks. Optionally, `editMessageText` updates the message with the admin’s decision and a timestamp.  
  2. The standard `Answer Callback Query` stops the loading spinner on Telegram.
- The admin’s choice (approved/rejected) is written to the `status` column via an **Update Row** node.
- Another row lookup retrieves the full record, and a final IF checks: if `status` includes `"Approved"` and `welcome_email_sent` is `empty`, a **Switch** on `source` sends either a welcome email (Gmail) or a welcome Telegram message to the original user, then sets `welcome_email_sent = "Sent 👍🏻"`.

### 4. Asynchronous Sweeper (Lead Recovery Engine)
- **Trigger:** A Schedule Trigger runs every 5 minutes.
- **Data Retrieval:** An HTTP Request fetches the entire sheet (or all necessary columns) from the Google Sheets API.
- **Code Logic (JavaScript):**  
  1. For each row, if `Date.now() - created_at < 10 minutes`, the row is **skipped** entirely (race‑condition prevention).  
  2. For older rows:  
     - If **Non‑VIP** and `email_sent` is `empty` → action `send_email`.  
     - If **VIP** and `telegram_notified` is `empty` → action `send_telegram` (first attempt).  
     - If **VIP**, `status` is `pending`, and `telegram_notified` already contains a timestamp from a previous attempt, the code extracts that timestamp. If more than 10 minutes have elapsed since that timestamp, it issues action `send_telegram` with a `is_reminder` flag.  
     - If `status` includes `"Approved"` and `welcome_email_sent` is `empty` → action `send_welcome`.
- **Loop & Switch:** The resulting array is iterated (Loop) and routed via a Switch node. Each output branch performs the corresponding notification and then updates the relevant flag with a fresh `Date.now()` timestamp, preventing immediate re‑processing on the next Sweeper cycle.

---

## Stateful Flagging & Exactly-Once Semantics

| Flag | Initial Value | Updated After | Update Value |
|------|---------------|---------------|--------------|
| `email_sent` | `empty` | Non‑VIP email (or Telegram) sent | `"Sent 👍🏻" + Date.now()` |
| `telegram_notified` | `empty` | VIP admin alert sent (first or reminder) | `"Sent 👍🏻" + Date.now()` |
| `welcome_email_sent` | `empty` | Welcome email/message after admin approval | `"Sent 👍🏻"` |
| `status` | `pending` | Admin clicks Approve/Reject | `"Approved👍🏻"` / `"Rejected👎🏻"` |
| `created_at` | `Date.now()` | Never modified | `Date.now()` |

Before every notification action (in both main workflow and Sweeper), a row lookup retrieves the current flag value. The notification only proceeds if the flag is exactly `empty` (or, for VIP reminders, if the timestamp in `telegram_notified` is older than 10 minutes). Immediately after the notification succeeds, the flag is updated. This pattern ensures **exactly‑once** delivery even under retries, network blips, or concurrent executions.

---

## Error & Fault Tolerance

- **AI Failure:** `On Error = Continue` on the AI Agent node, with a fallback `Set` node that marks the lead as `Ai_status = "Ai Failed"` and sets `lead_score = -1`. The pipeline continues.
- **Google Sheets Write Failure:** The Append Row node uses **Retry on Fail** (3 retries, 5s wait) and **On Error = Continue**. If all retries fail, a Telegram message is sent to the admin containing the error details.
- **Google Sheets API Call Failure (Dedup/Sweeper):** The HTTP Request nodes have `On Error = Continue`. The Dedup Code Node treats API errors conservatively (`is_new = false` to avoid duplicate appends). The Sweeper simply skips processing on failure.
- **Global Error Watchdog:** A completely separate workflow with an `Error Trigger` node listens for any hard stop in the main workflows. On trigger, it sends a detailed Telegram message to the admin (`workflow name`, `node name`, `error message`, `execution id`).

---

## Technologies & APIs

| Layer | Technology |
|-------|------------|
| Automation Engine | n8n (Community Edition, Docker) |
| Public Tunnel | Serveo (persistent subdomain) |
| Database | Google Sheets (REST API, OAuth2) |
| AI | OpenAI (or compatible) API |
| Messaging | Telegram Bot API (Webhook, Inline Keyboard, EditMessage) |
| Email | Gmail API |
| Monitoring | n8n Error Trigger + Telegram |

---

## Installation & Quick Start

1. **Clone this repo** and import the provided JSON workflow files into your n8n instance.
2. **Set up credentials** in n8n:
   - Google OAuth2 (for Sheets and Gmail) – create credentials on `localhost` first to avoid OAuth redirect issues.
   - Telegram Bot Token.
   - OpenAI API Key.
3. **Expose a public webhook URL** for Telegram using a tunnel (e.g., Serveo: `ssh -R f22mation:80:localhost:5678 serveo.net`). Set the `WEBHOOK_URL` Docker environment variable to `https://f22mation.serveousercontent.com`.
4. **Create a Google Sheet** with the required columns (see `schema.md` for the exact structure).
5. **Activate the three workflows**: Main, Sweeper, Error Watcher.
6. **Test** by sending a lead via Webhook (Postman) and via Telegram.

---

## Workflow Files
- `main_workflow.json` – Core lead processing and admin callback.
- `sweeper_workflow.json` – Scheduled recovery workflow.
- `error_watcher.json` – Global error notifier.

---

## License & Contact
MIT License.  
Built by **F22mation** – [GitHub](https://github.com/f22mation) | [LinkedIn](https://linkedin.com/in/f22mation) | [Linktree](https://linktr.ee/f22mation) | Email: f22mation@gmail.com
