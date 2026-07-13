# AI-Powered Lead Management & Automation System (n8n)

An enterprise-grade lead orchestration system built on n8n (self-hosted via Docker). This system automates the entire lifecycle of a lead: from multi-channel ingestion and AI-driven qualification to human-in-the-loop approval and automated follow-ups (Sweeper) for stuck tasks.

The system is designed with strict data integrity controls, including idempotency gates, rate-limit mitigation, and automated state reconciliation.

---

## 🏗️ System Architecture & Workflow Flow

The infrastructure consists of four interconnected sub-workflows designed to manage data persistence and communication channels seamlessly.

### 1. Ingestion & AI Processing Pipeline
* **Multi-Channel Input:** Receives incoming lead payloads via a RESTful `Webhook` (POST method) and a `Telegram Trigger` (incoming messages) simultaneously. 
* **User Experience:** The Telegram stream immediately triggers a `Send Receiving 🔔` node to acknowledge receipt to the end-user.
* **Data Standardization:** Data from both sources is normalized using a custom `unique` node and merged into a single execution stream via an `append` Merge node.
* **AI Qualification:** The payload passes through an `AI Agent` connected to the `OpenAI Chat Model` and a `Memory Tool`. It evaluates the lead, extracts a numerical score (`Lead_score`), and updates the `Ai Status`.
* **Fault Tolerance:** If the OpenAI API fails, the `Ai Failure ❌` node catches the error, logs the failure state, and routes the data back into the main pipeline via `Merge1` to prevent workflow disruption.
* **Data Cleansing & Deduplication:** The `Priority` node sanitizes the LLM output. The `Dedup Check` node verifies the lead against existing records using the `unique_id`, and the `If1` conditional node discards duplicates while passing new leads forward.

### 2. Routing, Ingestion & Idempotency Gates
* **Data Persistence:** Valid leads are appended to Google Sheets via the `Lead in sheet` node. If database insertion fails, an error trigger alerts the system administrator via `Sheet Error To Admin ❌`.
* **Segmented Routing (`IF VIP`):** Leads are split based on the calculated priority field:
  * **VIP Route (True Branch):** Fetches row details and processes them through an **Idempotency Gate** (conditional node) checking if `telegram_notified` is empty. If empty, it enters a `Loop Over Items`. A `Wait` node is placed inside the loop to mitigate **Telegram API Rate Limiting** during high-volume spikes. The administrator receives the notification via `Send a text message`, and the sheet row is updated via `Update row in sheet1`.
  * **Non-VIP Route (False Branch):** Fetches data via `Get row Non VIP`. The `IF None VIP` conditional node checks if notification columns are empty. If valid, `Switch1` routes the lead based on its `source` field:
    * *Output 0 (Email Source):* Sends a Gmail notification (`Receive 🔔`) and updates the row via `update nonvip`.
    * *Output 1 (Telegram Source):* Sends a Telegram message (`Send 🔔`) and updates the row via `update nonvip1`.

### 3. Human-in-the-Loop Decision Subsystem (Telegram Callback)
* **Trigger & Interaction:** Active when an admin clicks inline buttons (Approve/Reject) in Telegram. The `Telegram Trigger1` (`callback_query`) fires two parallel tracks:
  * *UI State Lock:* Invokes `markup edit` and `Edit a text message` to modify/disable the inline keyboard buttons, preventing duplicate clicks (Race Condition Prevention).
  * *Backend Feedback:* Fires `Answer Query a callback` to instantly dismiss the loading state on the admin's Telegram client.
* **State Update:** Updates the decision outcome in the `status` column via `Update row in sheet` and re-verifies the row data.
* **Final Notification:** The `IF APPROVED` node checks if the status equals approved. If true, `Switch2` routes the data based on the original `source` column (verifying that status flags are empty before sending):
  * *Output 0:* Sends a success email (`Success 🔔`) and records state in `VIP Sent`.
  * *Output 1:* Sends a success Telegram message (`Send 🔔 1`) and records state in `VIP Sent1`.

### 4. Asynchronous Automated Sweeper (Reconciliation Loop)
* **Time-Based Trigger:** A `Schedule Trigger` runs at defined periodic intervals to fetch active records via `HTTP Request2`.
* **Reconciliation Logic:** A `Code in JavaScript` node filters out leads whose `status` remains `pending` and where more than 10 minutes have elapsed since their creation (`created_at`) or their last notification timestamp.
* **Batch Processing:** Data loops through `Loop Over Items1` into a 4-state routing `Switch` node:
  * *Output 0 (Stuck Email):* If the email notification column is empty, it fires a follow-up email (`Receive 🔔 Sweeper`) and updates the sheet via `update Sweeper emailsent`.
  * *Output 1 (Stuck Telegram / Expired Pending Status):* If the lead source is Telegram, or if the `telegram_notified` column is marked `sent` but the lead has been abandoned in a `pending` state for over 10 minutes, it re-routes the task back to the admin's Telegram interface (`Send Sweeper`). The database is updated via `Update Sweeper`, appending the current system timestamp (`Date.now()`).
  * *Output 2 (Stuck VIP Follow-up):* Sends a dedicated VIP recovery email (`Success 🔔 Sweeper`) and records state via `VIP Sent Sweeper`.

---

## 📊 Google Sheets Database Schema

The architecture relies on the following structural layout within the spreadsheet database:

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `name` | String | Lead's full name |
| `email` | String | Lead's primary email address |
| `company` | String | Organization or company name |
| `budget` | Numeric | Associated project/lead budget |
| `message` | String | Raw text message or project description |
| `unique_id` | String | Generated unique identifier used for deduplication |
| `Ai Status` | String | Logged processing status of the OpenAI API |
| `status` | String | Overall lead lifecycle state (`pending`, `approved`, `rejected`) |
| `Lead_score` | Numeric | Numerical score evaluated by the AI Agent |
| `source` | String | Ingestion channel origin (`webhook`, `telegram`) |
| `chat_id` | String | Dedicated Telegram chat identifier for routing |
| `created_at` | Timestamp | System creation date and time |
| `priority` | String | Normalized priority rating derived from the AI score |
| `telegram_notified` | String | State tracking flag concatenated with `Date.now()` timestamps |

---

## 🛠️ Tech Stack

* **Workflow Orchestration:** n8n (Self-Hosted / Docker Infrastructure)
* **Artificial Intelligence:** OpenAI API (`GPT-4o` Core Engine)
* **Data Persistence:** Google Sheets API
* **Communication layers:** Telegram Bot API, Gmail/SMTP Services
* **Runtime Logic:** JavaScript (ES6+ for data filtering & execution timing)

---

## ⚙️ Deployment & Setup

### Prerequisites
* Docker and Docker Compose installed on a host environment.
* Google Cloud Console account with Google Sheets API credentials enabled.
* Telegram Bot API Token obtained via `@BotFather`.
* OpenAI API account with an active API Key.

### Installation Steps
1. **Clone Workspace Configuration:** Clone your n8n workspace directory or spin up your self-hosted instance using Docker Compose.
2. **Import Blueprints:** Create a new workflow in your n8n panel, open the top-right menu, select **Import from File**, and upload the respective JSON files for the Main, Callback, and Sweeper workflows.
3. **Environment Credentials:** Configure and link your credentials within the n8n credential manager for:
   * Google Sheets OAuth2
   * OpenAI API
   * Telegram Bot API
   * Gmail OAuth2 / SMTP
4. **Database Initialization:** Ensure your target Google Sheet matches the column structure detailed in the [Database Schema](#-google-sheets-database-schema) section.
5. **Webhook Mapping:** Copy the production webhook URL from your Ingestion workflow and paste it as the target URL in your external lead-generation forms or platform configurations.
