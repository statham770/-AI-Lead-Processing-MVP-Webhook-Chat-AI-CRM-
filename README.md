# 🎯 AI Lead Processing MVP (Webhook + Chat → AI → CRM)

Automated lead nurturing, qualification, and routing system powered by **n8n**, local artificial intelligence (**Ollama**), and integration with **Google Sheets** and **Telegram**.

This repository contains a ready-to-use n8n workflow configuration file, deployment instructions, and a demonstration video of the system in action.

---

## 🏗️ 1. Workflow Logic and Architecture Description

The system is designed using a modular approach to ensure maximum data processing stability, regardless of the request source. It unifies two independent lead generation channels into a single analytical hub.

### 🔄 Data Flow Diagram:

```
[ Webhook Trigger ] ──┐
                      ├──> [ Data Normalization ] ──> [ AI Processing (Ollama) ] ──> [ Parse LLM Response ]
[ Chat Trigger ] ─────┘                                                                        │
                                                                                               ▼
[ Chat Response ] <── [ Telegram Node ] <── [ Format Telegram Message ] <── [ Google Sheets Node ]
```

### 📋 Detailed Node Breakdown:

| № | Node Name | Type / Engine | Business Logic and Function Description |
|---|---|---|---|
| **1a** | `Webhook` | Webhook Trigger | Accepts raw `POST` requests from external sources (landing pages, lead forms, third-party CRMs, chatbots). |
| **1b** | `Chat Trigger` | LangChain Chat Trigger | Interface node for the built-in n8n chat, allowing direct interaction with the AI via the web panel. |
| **2** | `Data Normalization` | Code (JavaScript) | **Critical node.** Standardizes data from both triggers into a single `normalized_payload` object. This isolates downstream logic from changes in incoming request structures. |
| **3** | `AI Processing` | LangChain Chain | Sends the normalized request text to the language model using a structured system prompt. |
| **4** | `Ollama Model` | Chat Ollama | Local AI engine. Uses a model (e.g., `qwen3.5:9b` or `gemma`), operating at a low temperature (`0.1`–`0.2`) to minimize hallucinations and ensure accurate entity extraction. |
| **5** | `Parse LLM Response` | Code (JavaScript) | **Logical cleaning hub.** Uses regex to extract clean JSON from the AI response (even if the model wrapped it in markdown). Restores the original request text to prevent `undefined` errors and automatically adds timestamps. |
| **6** | `Google Sheets` | Google Sheets | Acts as the CRM/Database. Automatically maps AI-cleaned fields to corresponding spreadsheet columns (Name, Company, Budget, Lead Temperature, etc.). |
| **7** | `Format Telegram Message` | Code (JavaScript) | Generates a structured, emoji-rich report for the sales team (formats bold text, italics, and proper spacing). |
| **8** | `Telegram` | Telegram Node | Sends the generated report in real-time to your internal Telegram channel or working group. |
| **9** | `Chat Response` | Code (JavaScript) | Returns a final, clean text string to the `Chat Trigger` interface (*"Thank you! Your request has been received..."*), completely hiding technical logs from the user. |

---

## 🛠️ 2. AI Analysis Specification (Prompting & Parsing)

To qualify a lead, the model analyzes the text and extracts 6 key parameters:

- **`summary`** — Brief summary of the request in Ukrainian.
- **`lead_temperature`** — Assessment of the client's readiness to purchase: `Hot` (ready to buy right now), `Warm` (shows interest, comparing options), `Cold` (just asking questions).
- **`type`** — Market segment: `B2B` (business-to-business) or `B2C` (business-to-consumer).
- **`company`** — Organization name (if mentioned).
- **`phone`** — Contact phone number.
- **`budget`** — Financial boundaries of the request (e.g., *"3000 dollars"*).

Thanks to the script in `Parse LLM Response`, the system is protected against failures: if the local model cannot find a phone number or budget, it automatically assigns the value `"Not specified"`, without breaking the database structure.

---

## 🚀 3. Setup and Deployment Instructions

### 📋 Prerequisites

Before importing the workflow, make sure you have the following components deployed and configured:

1. **n8n** (locally via Docker or in the n8n Cloud).
2. **Ollama** (installed on the same server/PC where n8n is running, or accessible over the network).
3. A downloaded model in Ollama (recommended: `qwen3.5:9b`, `gemma2`, or `llama3`). Download command:
   ```bash
   ollama run qwen3.5:9b
   ```
4. **Telegram bot** created via [@BotFather](https://t.me/BotFather) (you will need the *Bot Token* and the target channel's *Chat ID*).
5. **Google Sheet** with access configured via the Google Cloud Console (OAuth2 or Service Account).

---

### ⏱️ Step-by-Step Workflow Configuration

#### Step 1. Import the scenario into n8n
1. Create a new empty workflow in n8n.
2. Click on the menu (three dots in the top right corner) → **Import from File**.
3. Select the configuration file `Lead Processing MVP (Webhook + Chat -> AI -> CRM).json` from this repository.

#### Step 2. Ollama Configuration
1. Open the **Ollama Model** node.
2. In the **Base URL** field, specify the address of your local Ollama server (usually `http://localhost:11434` or `http://host.docker.internal:11434` for Docker).
3. In the **Model Name** field, enter the name of the downloaded model (e.g., `qwen3.5:9b`).
4. *Important:* Make sure that the `Output Format: JSON` parameter is turned off (removed via the trash icon), as parsing is fully handled by the subsequent JavaScript node.

#### Step 3. Google Sheets Preparation
1. Create a new Google Sheet.
2. In the **first row (Header Row)**, create columns with these exact names (case-sensitive):

   `leadSummary` | `leadTemperature` | `leadCategory` | `companyName` | `phoneNumber` | `budget` | `inquiryMessage` | `processedTime`

3. In n8n, open the **Google Sheets** node, select your Google account, specify the `Spreadsheet ID`, and choose the desired `Sheet`. The operation mode must be set to **Append row to sheet**.

#### Step 4. Telegram Notifications Configuration
1. Open the **Telegram** node.
2. Choose or create a new connection (Credentials) by pasting your bot's token.
3. In the **Chat ID** field, specify the ID of your group or channel (e.g., `-100XXXXXXXXXX`).
4. Ensure that the bot is added to this channel as an administrator with message-sending permissions.

#### Step 5. Activation
Click the **Save** button in the top right corner and flip the workflow toggle to **Active**.

---

## 📂 4. Repository Structure

```
├── Lead Processing MVP (Webhook + Chat -> AI -> CRM).json  # Workflow configuration file for import
├── README.md                                               # This document (instructions and logic overview)
└── demo.mp4                                                # System demonstration video
```

---

## 🛠️ 5. Troubleshooting

**Problem:** A message arrives in Telegram with the value `Inquiry: undefined`.
> *Solution:* Ensure you have updated the code in the `Parse LLM Response` node to the version that restores the original text via the `originalMessage` variable. The node must return the `inquiryMessage` key.

**Problem:** The n8n chat outputs a response as a JSON code block with curly braces `{}`.
> *Solution:* Check the `Chat Response` node. The final object must return a key named `text` (not `response`), because the n8n chat interface responds exclusively to the `text` marker to display a clean message.

**Problem:** The `AI Processing` node throws an `Unexpected end of JSON input` error.
> *Solution:* The local model is trying to return explanatory text along with the JSON, or the forced JSON format is enabled in the `Ollama Model` node, which truncates the generation. Disable the `Format: JSON` parameter in the `Ollama Model` node and lower the temperature to `0.1`.
