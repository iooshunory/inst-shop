# 🍄 MuhoMarket — AI Sales Chatbot with Operator Panel

> Multi-channel AI sales assistant for a natural mushroom & herbal products shop, built on n8n. Handles Facebook, Instagram, and web chat simultaneously — with smart escalation to human operators and a full Telegram-based CRM panel.

---

## 📌 What This Project Does

MuhoMarket is an end-to-end automated sales system for a Ukrainian online shop selling medicinal mushrooms, herbal teas, and natural remedies. The AI agent consults customers in authentic Ukrainian, recommends products based on their health needs, and hands off to a live operator when needed — all without the business owner manually handling every message.

**Core capabilities:**

- AI consultant running on Facebook Messenger, Instagram DMs, and embedded web chat simultaneously
- Retrieval-Augmented Generation (RAG) using a curated FAQ knowledge base and vector product catalog
- Smart product recommendations using semantic search across the full product range
- Real-time price and SKU lookup from KeyCRM and Google Sheets
- Article/resource sharing for customer education
- Automatic escalation to human operators with Telegram notifications
- Telegram-based operator CRM panel for managing leads, taking conversations, and re-enabling the agent

---

## 🏗️ Architecture Overview

```
Incoming messages
  ├── Facebook Messenger  ──┐
  ├── Instagram DMs        ├──► Normalize → Buffer → Lock → AI Agent
  └── Web Chat (n8n)       ──┘              ↓
                                    PostgreSQL (leads, state)
                                             ↓
                                     AI Agent (GPT-4.1)
                                    ┌────────────────────┐
                                    │  Tools:            │
                                    │  • FAQ (RAG)       │
                                    │  • Vector Store    │
                                    │  • Product List    │
                                    │  • Articles        │
                                    │  • Think           │
                                    └────────────────────┘
                                             ↓
                              ┌──────────────────────────────┐
                              │  Escalation check            │
                              │  → Send reply to channel     │
                              │  → Notify operator (Telegram)│
                              └──────────────────────────────┘
```

---

## 🔧 Tech Stack

| Component | Technology |
|-----------|------------|
| Automation | n8n (self-hosted) |
| AI Model | GPT-4.1 (main agent), GPT-4.1-mini (search & FAQ) |
| Vector Store | Supabase pgvector |
| Database | PostgreSQL (leads, buffers, locks, operators) |
| CRM | KeyCRM API |
| Product Catalog | Google Sheets |
| Channels | Facebook Graph API v22, Instagram Graph API v22/v23 |
| Operator Panel | Telegram Bot API |
| Embeddings | OpenAI text-embedding-ada |

---

## 📁 Workflow Files

| File | Description |
|------|-------------|
| `main-chatbot.json` | Core pipeline: intake, buffering, AI agent, reply routing, escalation logic, operator notifications |
| `operator-panel.json` | Telegram bot for operators: lead lists, pagination, take/finish conversations, manual agent control |
| `product-search.json` | Sub-workflow: semantic product search across KeyCRM + Google Sheets using GPT-4.1-mini |
| `sub-faq-rag.json` | Sub-workflow: RAG retrieval from Supabase FAQ knowledge base |
| `vector-store-setup.json` | One-time setup: loads articles and FAQ Q&A pairs from Google Drive into Supabase vector store |

---

## 🤖 AI Agent Logic

The agent follows a strict step-by-step sales script defined in the system prompt:

1. **Greeting** — first contact or 10+ hours since last message
2. **Experience check** — "Have you used [product] before?"
3. **Needs clarification** — "What's bothering you? What effect are you looking for?"
4. **Recommendation** — 2-3 best products across categories (mushrooms, herbs, teas, ointments), always paired with educational articles
5. **Forms & pricing** — shown only when asked, pulled from live CRM data
6. **Order collection** — triggered by buy signals, collects name/phone/city/post office number

**Tool call order per message:**
1. `FAQ` — always first, retrieves similar past conversations and suggested tone
2. `Vector Store` — semantic product property search
3. `articles` — retrieves Telegram article links by product name
4. `list_of_products` — price and SKU verification from KeyCRM/Sheets

---

## 🔁 Message Buffering & Locking

To handle rapid consecutive messages from one user without triggering multiple agent responses:

1. Each incoming message is inserted into a `chat_buffer` table
2. The workflow attempts to acquire a `chat_locks` row for the `chat_id`
3. If a lock already exists → message queued in buffer, execution stops
4. If lock acquired → wait, collect all buffered messages, concatenate, process once
5. After agent response → delete buffer and lock

This prevents duplicate AI responses when customers send multiple short messages.

---

## 👨‍💻 Operator Panel (Telegram)

Operators manage all leads through an inline-button Telegram bot. Commands and flows:

| Action | Description |
|--------|-------------|
| `/lead` | Opens main menu with 3 lead categories |
| 🤖 Agent active | Leads chatting with bot in last 2 hours |
| 👤 Waiting for operator | Leads that triggered escalation |
| 🔴 Manually disabled | Leads where agent was turned off |
| ✋ Take client | Assigns operator, disables agent (`temporary`) |
| 🏁 Finish conversation | Releases lead, re-enables agent |
| 🔴 Disable manually | Sets `agent_disable_type = manual` permanently |
| 🟢 Enable agent | Re-activates bot for the lead |
| 🔄 Refresh | Reloads current list with updated data |

Lead cards show: channel, name/username, agent status, operator status, last message, timestamps.

**Auto-disconnect:** A scheduled job runs every 30 minutes and releases any operator-connected leads that have had no activity for 5+ hours. Operators are notified via Telegram.

---

## ⚡ Escalation Triggers

The agent automatically escalates (sets `waiting_for_operator = true`, notifies all operators) when:

- Customer says "get me a human" or similar
- Serious diagnosis mentioned (cancer, metastasis, stage 4)
- Customer has already provided shipping details
- Agent hits max iterations
- Response contains order collection phrase
- Topic falls outside shop scope

---

## 🗄️ Database Schema (PostgreSQL)

**`leads`** — main table

```sql
external_id          VARCHAR   -- platform user ID
channel_type         VARCHAR   -- facebook / instagram / chat
username             VARCHAR
first_name           VARCHAR
waiting_for_operator BOOLEAN
operator_connected   BOOLEAN
connected_operator_id    INT
connected_operator_name  VARCHAR
agent_disabled       BOOLEAN
agent_disable_type   VARCHAR   -- temporary / manual / NULL
last_activity_at     TIMESTAMP
last_message_from    TEXT
channel_id           VARCHAR
```

**`chat_buffer`** — temporary message queue per chat  
**`chat_locks`** — mutex locks per chat_id  
**`operators`** — Telegram operator accounts with IDs

---

## 📦 Vector Store Structure

Two separate document collections in Supabase, filtered by metadata:

| Collection | Metadata tag | Contents |
|-----------|-------------|----------|
| Knowledge base | `FAQ` | Q&A pairs from real customer conversations |
| Articles | `articles` | Telegram article links with product context |

FAQ documents are parsed from a structured `Question:/Answer:` text file. Articles are Google Docs from a Drive folder, chunked and indexed automatically.

---

## 🔑 Required Credentials

To deploy this system you will need:

- OpenAI API key
- Supabase project URL + service key
- PostgreSQL connection string
- Facebook App with Messenger permission + Page Access Token
- Instagram Business account + Graph API token
- KeyCRM API key
- Google OAuth2 (Drive + Sheets)
- Telegram Bot token

All credential placeholders in the JSON files are marked as `YOUR_*_CRED_ID` / `YOUR_*_TOKEN`.

---

## 📊 Key Metrics This System Addresses

- **Response time:** instant 24/7 vs. manual response delays
- **Operator load:** only escalated conversations require human attention
- **Consistency:** every customer gets the same structured sales flow
- **Multi-channel:** one logic layer serves Facebook, Instagram, and web chat
