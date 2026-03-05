# 🛍️ Belle — AI Personal Buyer

Belle is an AI-powered e-commerce shopping assistant built on Telegram. It helps users find products from a catalog using natural language text or by uploading images. Belle remembers returning users and personalises recommendations based on their shopping history.

> 💬 **Try it live:** Search `@n8n_shopping` on Telegram

---

## Demo

| Text Search | Image Search | Memory |
|---|---|---|
| "find me cool t-shirts" | Send a photo of any clothing item | Belle greets you by name on return visits |
| Returns product photos with prices | Gemini Vision extracts keywords | Tracks favourite categories and style |
| Prices converted from INR to USD | Finds visually similar items in catalog | Logs session type and last search |

---

## Architecture

```
User (Telegram)
      │
      ▼
Telegram Trigger (n8n on Railway)
      │
      ▼
Save Chat ID
      │
      ▼
If: Has Photo?
 ├── YES → Extract image prompt → Download photo → Gemini Vision analysis → Extract keywords
 └── NO  → Pass text directly
      │
      ▼
AI Agent (Gemini 2.0 Flash)
 ├── Tool: Read Memory (Google Sheets)
 ├── Tool: Search Catalog (Supabase Vector Store)
 └── Tool: Write Memory (Google Sheets)
      │
      ▼
Split products into items
      │
      ▼
If: Has Products?
 ├── YES → Send product photos (Telegram) → Send assistant message
 └── NO  → Send assistant message only
```

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Workflow orchestration | [n8n](https://n8n.io) | Visual workflow automation |
| Hosting | [Railway](https://railway.app) | Cloud deployment for n8n |
| LLM | Google Gemini 2.0 Flash | Conversation + product reasoning |
| Vision | Google Gemini 2.5 Flash | Image analysis for visual search |
| Vector database | [Supabase](https://supabase.com) | Product catalog + semantic search |
| Embeddings | HuggingFace `all-MiniLM-L6-v2` | Product vector embeddings |
| Memory | Google Sheets | Persistent user preference storage |
| Interface | Telegram Bot API | Chat interface |

---

## Features

**🔍 Text-based product search**
Users describe what they're looking for in natural language. The AI searches the vector catalog and returns the most relevant products with photos, prices, and a personalised reason for each match.

**📸 Image-based product search**
Users upload any product photo. Gemini Vision analyzes the image to extract detailed keywords (category, color, material, style, demographic) which are then used to search the catalog for visually similar items.

**🧠 Persistent user memory**
Belle remembers every user across sessions using Google Sheets as a lightweight database. It tracks name, favourite category, price range, style preference, session type, and interaction count. Returning users are greeted by name and receive personalised suggestions.

**💬 Natural conversation**
Belle handles general conversation gracefully — greetings, capability questions, and small talk — without triggering unnecessary product searches.

---

## Memory Schema

Stored in Google Sheets with the following columns:

| Column | Description |
|---|---|
| `chat_id` | Unique Telegram user identifier |
| `name` | User's name if shared in conversation |
| `session_type` | `text_search`, `image_search`, or `general_chat` |
| `last_search` | Most recent search query |
| `favorite_category` | Inferred preferred product category |
| `price_range` | Budget preference if mentioned |
| `style_preference` | Inferred style (casual, sporty, formal, etc) |
| `total_interactions` | Running count of conversations |
| `notes` | Freeform agent observations |
| `last_seen` | Timestamp of last interaction (YYYY-MM-DD HH:mm:ss) |

---

## Product Catalog

- Source: Flipkart e-commerce dataset
- Stored in Supabase with pgvector extension
- Embeddings generated using `sentence-transformers/all-MiniLM-L6-v2`
- Semantic search via custom `match_products` SQL function
- Prices displayed in USD (converted from INR at 1 USD = 83 INR)

## Dataset

**Source:** [Flipkart Products Dataset](https://www.kaggle.com/datasets/PromptCloudHQ/flipkart-products) via Kaggle (PromptCloudHQ)

The original dataset contains ~20,000 Flipkart product listings. For this project:

- Randomly sampled **300 products** from the full dataset
- Preserved the following columns:

| Column | Description |
|---|---|
| `uniq_id` | Unique product identifier |
| `product_name` | Full product title |
| `product_category_tree` | Category hierarchy (e.g. Clothing >> Men >> T-Shirts) |
| `retail_price` | Original price in INR |
| `image_url` | Hosted product image URL |
| `image_local_path` | Local path reference |
| `text` | Combined text field used for embedding generation |

- Cleaned null values and malformed rows
- Generated vector embeddings using `sentence-transformers/all-MiniLM-L6-v2` on the `text` column
- Uploaded to Supabase with pgvector for semantic similarity search
- Prices displayed in USD at checkout (converted at 1 USD = 83 INR)
---

## How to Run It Yourself

### Prerequisites
- [n8n](https://n8n.io) instance (self-hosted or cloud)
- [Supabase](https://supabase.com) project with pgvector enabled
- Google Cloud project with Gemini API and Google Sheets API enabled
- Telegram bot token from [@BotFather](https://t.me/botfather)
- HuggingFace API key

### Setup Steps

**1. Clone this repo**
```bash
git clone https://github.com/Beier-H/shopping-agent-telegram
```

**2. Import workflow into n8n**
- Open your n8n instance
- Click **+** → **Import from file**
- Select `workflow.json`

**3. Add credentials in n8n**
- Google Gemini API key
- Supabase URL + anon key
- Telegram bot token
- HuggingFace API key
- Google Sheets OAuth

**4. Set up Supabase**
- Enable pgvector extension
- Create `products` table
- Create `match_products` function for vector similarity search
- Upload your product catalog and generate embeddings

**5. Set up Google Sheets memory**
Create a sheet with columns:
```
chat_id | name | session_type | last_search | favorite_category | price_range | style_preference | total_interactions | notes | last_seen
```

**6. Update workflow**
- Replace Google Sheets URL in both memory nodes
- Replace bot token in the image download Code node
- Activate the workflow

**7. Test**
Search your bot on Telegram and send a message.

---

## Project Structure

```
belle-shopping-bot/
├── README.md
└── workflow.json        # n8n workflow — import this into your n8n instance
```

---

## Design Decisions

**Why n8n?**
n8n provides visual workflow orchestration that makes the pipeline easy to understand, modify, and demo. It handles webhook management, credential storage, and node chaining out of the box.

**Why Telegram over a web app?**
Telegram provides a native mobile experience with zero frontend maintenance. The bot is always accessible, requires no login, and feels natural for a conversational shopping assistant.

**Why Google Sheets for memory?**
Google Sheets is transparent, easily auditable, and requires no additional database setup. For a hiring demo it also makes it easy to show the hiring team exactly what data is being stored and how the memory system works in real time.

**Why Supabase + pgvector?**
Vector similarity search enables semantic product matching — users don't need to use exact product names. "something cozy for winter" finds relevant results even if no product is literally named that.

---

## Limitations

- Product catalog is from Flipkart India — some images may be broken or unavailable
- Gemini free tier has rate limits (10 requests/min) — may be slow under heavy load
- Memory is per-user but not per-conversation — context resets between n8n executions
- Image search accuracy depends on Gemini Vision keyword extraction quality

---

## Author

Feel free to reach out with any questions about the implementation.
