# SYNTAX-AI---shop-assistant

[ai-telegram-bot-case-study-en.md](https://github.com/user-attachments/files/30715540/ai-telegram-bot-case-study-en.md)
# AI Business Assistant on Telegram

**An automated chatbot with long-term memory, a knowledge base, and error handling — built with n8n and the Groq LLM**

---

## The Problem

Build a Telegram bot that can take over part of a support team's workload: answering customers' everyday questions (shipping, payment, returns, product availability) quickly, around the clock, in the customer's own language, without human involvement.

## The Core Problem It Solves

Most simple AI bots lose their memory after a server restart and don't ground their answers in up-to-date business facts — they answer "from imagination," which leads to hallucinations and inaccurate information. This bot solves both problems.

## Features

- **Natural-language conversation** — customers write questions the way they'd talk to a person, and the bot understands context and answers directly
- **Automatic language detection** — the bot replies in whatever language the customer writes in, with no manual configuration
- **Long-term memory** — chat history is stored in Google Sheets and survives server restarts; the bot pulls the last 15 messages so it doesn't lose the thread of a conversation
- **Business knowledge base** — facts about the store (shipping, payment, returns, contact info) live in a separate sheet and are fed to the bot as a source of truth, rather than being invented by the model
- **`/reset` command** — customers can clear their chat history at any time and start fresh
- **Fault tolerance** — if an external service (the LLM or the spreadsheet) is temporarily unavailable, the bot doesn't hang; it tells the customer to try again shortly

## Architecture

```
Telegram Trigger
        ↓
       IF (/reset or a regular message?)
        │                                    │
   [/reset]                            [regular message]
        ↓                                    ↓
Delete chat history                 Log the message
        ↓                                    ↓
Confirm to customer               Read chat history (Google Sheets)
                                             ↓
                                  Read business knowledge base
                                             ↓
                                  Build context (history + facts)
                                             ↓
                                       AI Agent (Groq LLM)
                                             ↓
                                    Send reply to customer
                                             ↓
                                  Log the bot's response
```

## Tech Stack

| Component | Technology |
|---|---|
| Workflow orchestration | n8n |
| LLM | Groq (fast inference) |
| History & knowledge base storage | Google Sheets |
| Messaging channel | Telegram Bot API |
| Branching logic & error handling | n8n IF / Error handling |

## Sample Conversation

```
Customer: How long does delivery usually take?
Bot: Delivery usually takes 3–5 business days within the country.
     If you need it faster, we also offer express shipping
     (1–2 days) for an additional fee. Anything else I can help with?

Customer: /reset
Bot: Chat history cleared ✅
```

## Possible Extensions

- Connect to a real product/inventory database (instead of a static list of facts)
- Summarize older history to preserve long-term context without growing the prompt
- Notify the business owner on failures or on questions the bot couldn't answer
- Analytics on the most common customer inquiry topics

---

*Built as part of a hands-on AI-automation learning practice on n8n. The demo adapts to any business niche by swapping out the contents of the knowledge base.*
