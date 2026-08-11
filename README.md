[README (1).md](https://github.com/user-attachments/files/30949709/README.1.md)
# SYNTAX AI — Shop Assistant

**A multi-channel AI customer support assistant — Telegram bot and website chat widget — with long-term memory, a knowledge base, and error handling, built with n8n and the Groq LLM**

---

## The Problem

Build an AI assistant that can take over part of a support team's workload: answering customers' everyday questions (shipping, payment, returns, product availability) quickly, around the clock, in the customer's own language, without human involvement — and meet customers wherever they already are, whether that's Telegram or the business's own website.

## The Core Problem It Solves

Most simple AI bots lose their memory after a server restart and don't ground their answers in up-to-date business facts — they answer "from imagination," which leads to hallucinations and inaccurate information. This bot solves both problems.

## Features

- **Natural-language conversation** — customers write questions the way they'd talk to a person, and the bot understands context and answers directly
- **Automatic language detection** — the bot replies in whatever language the customer writes in, with no manual configuration
- **Long-term memory** — chat history is stored in Google Sheets and survives server restarts; the bot pulls the last 15 messages so it doesn't lose the thread of a conversation
- **Business knowledge base** — facts about the store (shipping, payment, returns, contact info) live in a separate sheet and are fed to the bot as a source of truth, rather than being invented by the model
- **Live order lookup (Tool)** — the AI Agent can query a Google Sheets "Orders" table in real time to answer "where is my order #101?" with the actual current status, instead of guessing
- **Live stock lookup (Tool)** — a second Tool checks a "Products" table for real-time stock availability, so the bot gives an honest answer instead of assuming an item is in stock
- **`/reset` command** — customers can clear their chat history at any time and start fresh
- **Fault tolerance** — if an external service (the LLM or the spreadsheet) is temporarily unavailable, the bot doesn't hang; it tells the customer to try again shortly
- **Hallucination guardrails** — the system prompt explicitly instructs the model never to invent details (contact info, tracking numbers, stock levels) that aren't present in the retrieved data, and the model's temperature is tuned down to keep answers grounded
- **Multi-channel** — the same AI Agent logic (memory, knowledge base, Tools, guardrails) is exposed through two channels: a Telegram bot and a floating chat widget embeddable on any website via a webhook, so the assistant can be deployed wherever a business's customers already are

## Architecture

Two independent trigger branches feed the same kind of pipeline (history → knowledge → AI Agent → Tools → reply), one per channel:

```
Telegram Trigger                        Website Chat Widget
        ↓                                        ↓
       IF (/reset or a                        Webhook (POST)
        regular message?)                         ↓
        │              │                  Log the message
   [/reset]      [regular msg]                     ↓
        ↓              ↓                  Read chat history (Google Sheets)
Delete history   Log the message                   ↓
        ↓              ↓                  Read business knowledge base
Confirm to      Read history                        ↓
customer         + knowledge                Build context (history + facts)
                        ↓                            ↓
                  Build context                AI Agent (Groq LLM) ← Tool: Order status
                        ↓                            │              ← Tool: Stock check
                AI Agent (Groq LLM) ← Tools           ↓
                        ↓                     Respond to Webhook (JSON)
                Send reply (Telegram)                ↓
                        ↓                     Widget renders reply on the page
                Log the bot's response
```

![Workflow overview](screenshots/08-workflow-overview.png)

## Tech Stack

| Component | Technology |
|---|---|
| Workflow orchestration | n8n |
| LLM | Groq (fast inference) |
| History & knowledge base storage | Google Sheets |
| Messaging channels | Telegram Bot API, custom website chat widget (HTML/CSS/JS) via Webhook |
| Branching logic & error handling | n8n IF / Error handling |

## Key Building Blocks

**`/reset` routing**
An IF node checks whether the incoming message is `/reset`. If so, the user's rows are deleted from the Google Sheets log and a confirmation is sent. Otherwise, the message flows into the normal reply pipeline.

![Reset command IF node](screenshots/02-if-node-reset.png)

**System prompt**
The AI Agent's system message combines the business knowledge base and the last 15 messages of chat history, with instructions to reply in the customer's language and stay grounded in the provided facts.

![AI Agent system prompt](screenshots/03-ai-agent-config.png)

**Persistent chat log**
Every user message and bot reply is appended as its own row in Google Sheets, giving a full, timestamped conversation history that survives restarts.

![Google Sheets log](screenshots/04-google-sheets-log.png)

**Tools — giving the agent real actions, not just text**
Two Google Sheets Tools are connected to the AI Agent so it can look up live data instead of relying only on its static system prompt:
- *Order status lookup* — filters an "Orders" sheet by `order_id`, with the ID supplied dynamically by the model via `$fromAI()`
- *Stock availability lookup* — filters a "Products" sheet by product name the same way

The model decides on its own, based on each Tool's description, whether a customer's question requires calling one of them.

![AI Agent Tools configuration](screenshots/06-ai-agent-tools.png)

**Debugging a real hallucination issue**
While testing, the bot occasionally invented plausible-sounding details that weren't in any data source — for example, a support phone number and email address that didn't exist anywhere in the knowledge base. Diagnosing it meant checking, execution by execution, whether a Tool had actually been called (it hadn't) and confirming the fabricated data wasn't present anywhere in the sheets. The fix combined two changes: adding an explicit instruction to the system prompt telling the model to say "I don't have that information" instead of inventing an answer, and lowering the Groq model's temperature to reduce its tendency to fill gaps creatively.

**A second channel: website chat widget**
To make the same assistant deployable on a business's own site (not every customer wants to open Telegram), the workflow gained a second entry point: a `Webhook` node in place of the Telegram Trigger, feeding the same kind of history → knowledge → AI Agent → Tools pipeline, and a `Respond to Webhook` node that returns the reply as JSON. On the front end, a small floating chat bubble (plain HTML/CSS/JS, no framework) opens a chat window and calls the webhook on each message. Getting this working end-to-end surfaced a few real integration issues: a CORS policy blocking the site's requests (fixed with an `Access-Control-Allow-Origin` response header), a `Respond to Webhook` node that looked connected on the canvas but wasn't actually wired to send its response until the Webhook node's "Respond" setting was switched to "Using Respond to Webhook Node," and a JSON body that broke because the bot's raw reply text (with its own quotes and line breaks) was pasted directly into a hand-written JSON string instead of being safely serialized as an expression object.

![Website chat widget](screenshots/07-website-widget.png)

## Try It

The website widget is live on a demo storefront page, built and hosted for free on GitHub Pages:

**[qweghtytest.github.io](https://qweghtytest.github.io)** — click the chat bubble in the bottom-right corner.

## Sample Conversation

Real tested conversations from the Telegram bot, showing multi-language support, the `/reset` command, and both Tools in action:

![Shipping and returns test](screenshots/telegram_conversation-1.jpg)
![Stock check and reset test](screenshots/telegram_conversation-2.jpg)
![Payment and multilingual test](screenshots/telegram_conversation-3.jpg)
![Combined Tools test](screenshots/telegram_conversation-4.png)

## Possible Extensions

- Add more Tools (e.g. initiating a return, checking delivery ETAs) as the business niche demands
- WhatsApp Business as a third channel, reusing the same AI Agent pipeline
- Summarize older history to preserve long-term context without growing the prompt
- Notify the business owner on failures or on questions the bot couldn't answer
- Analytics on the most common customer inquiry topics

---

*Built as part of a hands-on AI-automation learning practice on n8n. The demo adapts to any business niche by swapping out the contents of the knowledge base.*
