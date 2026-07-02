# LinkedIn AI Topic Ranker (n8n)

An n8n workflow that scores yesterday's collected topics using a Gemini-powered AI agent, picks the best one, and hands it off to the Content Generator workflow.

Runs daily at 8:10 AM, ten minutes after the [Trend Collector](https://github.com/Chirag-is-a-dev/linkedin-trend-collector-n8n) populates the `topics` table.

## What it does

1. **Scheduled trigger** — runs every day at 8:10 AM (`10 8 * * *`).
2. **Fetches unscored topics** — pulls the latest 50 topics from Postgres where `used = false AND score = 0`.
3. **Formats for AI** — builds a numbered topic list for the LLM prompt.
4. **AI Agent scoring** — a LangChain agent (Google Gemini) scores every topic 1–10 for LinkedIn post potential, based on:
   - Current relevance to AI/automation trends
   - Interest to n8n developers and builders
   - Practical teaching value
   - Discussion/debate potential
   - Relevance to automation, LLMs, or workflow tools
   - Output is enforced as a JSON array (`{"index": n, "score": n}`) via a **Structured Output Parser**, with an **Auto-fixing Output Parser** as a fallback if the model's output doesn't parse cleanly.
5. **Parses scores** — a Code node maps AI scores back to the original topics and sorts them descending. Handles both the parsed-array format n8n 2.25.7 returns under the `output` key and the older `choices[0].message.content` / `message.content` formats, for compatibility.
6. **Updates Postgres** — writes each topic's score back to the `topics` table.
7. **Selects the winner** — grabs the highest-scoring unused topic and marks it `selected = true`.
8. **Notifies via Telegram** — sends a formatted message announcing the winning topic and its score, with a note that Content Generator kicks off at 8:15 AM.

## Requirements

- n8n instance with LangChain nodes enabled (`@n8n/n8n-nodes-langchain`)
- Postgres database with the `topics` table (see [Trend Collector](https://github.com/Chirag-is-a-dev/linkedin-trend-collector-n8n) for schema)
- Google Gemini API key (via `googlePalmApi` credential in n8n)
- Telegram bot + chat ID for notifications

## Setup

1. Import `topic-ranker.json` into n8n.
2. Add your Postgres credentials to all four Postgres nodes (Fetch Unscored Topics, Update Scores, Get Winning Topic, Mark as Selected).
3. Add your Gemini API key to both **Google Gemini Chat Model** nodes.
4. Replace `YOUR_TELEGRAM_CHAT_ID` in the **Notify Winner** node with your actual Telegram chat ID, and attach your Telegram bot credentials.
5. Activate the workflow. It should run after (not before) the Trend Collector job.

## Notes

- The AI Agent uses a two-stage output parser: a **Structured Output Parser** defines the expected JSON schema, and an **Auto-fixing Output Parser** wraps it to self-correct malformed LLM output before it reaches the Code node.
- The `Parse Scores` Code node is defensive by design — it checks three possible response shapes because n8n's OpenAI/LLM node output format has varied across versions.
- Only the single top-scoring topic per run gets `selected = true`; the rest keep their scores for future reference or re-ranking.

---
Built and maintained by [FlowForge AI](https://flowforge-ai-agents.web.app).
