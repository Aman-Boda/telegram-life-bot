# Life Bot — Expense & Task Tracker (Telegram + AI)

A Telegram bot that understands plain-language messages — "spent 200 on lunch," "add task buy groceries," "done 2" — and automatically logs expenses, manages a task list, and sends a daily summary, all through natural conversation rather than commands or buttons.

## What it does

Message the bot normally, and it figures out what you mean:

- **"spent 200 on lunch"** → logs an expense and confirms the amount
- **"add task buy groceries"** → adds a task with a short, readable ID
- **"show my tasks"** → lists everything still open, by ID
- **"done 2"** or **"done buy groceries"** → marks that task complete, matching by ID or by text
- **Anything unrecognized** → a helpful fallback reply instead of silence

Separately, a scheduled job runs once a day and messages you a summary: today's spending, an all-time total, and every open task — without you asking for it.

## Architecture

```
Telegram message → Telegram Trigger → Build Classifier Request → Classify (Gemini) → Parse Classification
                                                                                             │
                                                                                      Route by Type
                        ┌───────────────┬──────────────────┬──────────────────┬─────────────┤
                      Expense         Task_add           Task_list        Task_complete   Fallback
                        │               │                  │                  │             │
                   Log Expense    Calculate Next ID    Get Open Tasks    Get All Tasks   Send Fallback
                        │               │                  │                  │             Reply
                   Send Reply      Log Task          Format & Reply    Find Matching Task
                                        │                                      │
                                   Send Reply                            Task Found?
                                                                        ┌────┴────┐
                                                                       Yes        No
                                                                        │          │
                                                                  Update Status  Send "not found"
                                                                        │
                                                                   Send Reply

Separately:
Schedule Trigger (daily) → Get Expenses → Get Tasks → Build Summary → Send Summary
```

## Tech stack

- **[n8n](https://n8n.io)** — self-hosted workflow engine
- **Telegram Bot API** — the chat interface (created via @BotFather, no business verification needed, unlike WhatsApp)
- **Google Gemini API** — classifies each message into an intent (expense / add task / list tasks / complete task / other) and extracts the relevant data
- **Google Sheets API (OAuth2)** — two tabs, Expenses and Tasks, acting as the database
- **ngrok** — exposes the local n8n instance over HTTPS for local development, since Telegram (unlike a basic webhook you call yourself) requires a real public HTTPS address to deliver messages to

## Technical challenges solved

This was built with real debugging along the way, not a clean first pass — these are the actual bugs and what they taught:

- **Telegram needs a public HTTPS URL, even for local development.** Unlike calling your own webhook from a script, Telegram's servers have to reach *your* server — `localhost` is rejected outright. Solved with ngrok as a tunnel, plus n8n's `WEBHOOK_URL` environment variable so n8n advertises the tunnel's address instead of localhost.
- **Testing via "Execute Workflow" caused 4x duplicate runs.** Telegram retries message delivery if it doesn't get a fast acknowledgment, and the temporary test listener was too slow. Publishing the workflow (a properly running listener) fixed it — fast is a correctness requirement here, not just a performance nice-to-have.
- **Data silently disappears between nodes.** Every action node (an HTTP call, a Sheets write) replaces the data flowing through it with its own output — anything not explicitly carried forward is gone. `chat_id`, captured early from the incoming message, kept vanishing a few nodes later. The fix, applied consistently: reference the originating node explicitly (`$('Node Name').item.json.field`) rather than assuming data survives in `$json`.
- **`.item` silently grabs the wrong item once multiple items are in flight.** A Sheets read returning several rows meant a downstream Code node's `$('Parse Classification').item` had to *guess* which one to pair with — and guessed wrong, pulling in a different task entirely. `.first()` fixed it by removing the ambiguity: this value only ever exists once per message, so always take the first (and only) one, regardless of how many other items are present.
- **A condition that looks correct on screen, but isn't.** The exact same category of bug surfaced three separate times across this build (a boolean comparison, a Switch node rule, an IF node) — each one visually identical to a working condition, yet routing incorrectly. Re-typing the condition never fixed it; only deleting and recreating it from scratch did, each time. The underlying cause stayed invisible, but the fix pattern became reliable: when a condition that *looks* right keeps misbehaving, stop editing it and rebuild it.
- **Matching rows by position (`row_number`) is unreliable.** Early design matched tasks for completion by their row number in the sheet — a pattern with known, documented reliability issues in n8n's community. Switched to generating a real, short, sequential ID for each task at creation time, and matching on that instead — a stable identifier instead of a position that can shift.
- **An unwanted footer on every message.** n8n's Telegram (and Email, Slack) nodes append a line like *"This message was sent automatically with n8n"* by default — not something typed in error, but a built-in setting. Turned off via each node's "Append n8n attribution" option.

## Setup

1. Create a bot via Telegram's **@BotFather** (`/newbot`) and copy the API token
2. Create a Google Sheet with two tabs: **Expenses** (Timestamp, Amount, Description, Raw Message) and **Tasks** (Timestamp, Task, Status, Completed At, ID)
3. Import `workflow.json` into n8n
4. Create credentials: Telegram (the bot token), Gemini (Header Auth), Google Sheets OAuth2
5. For local development, run [ngrok](https://ngrok.com) pointed at your n8n port, and set the `WEBHOOK_URL` environment variable to your ngrok HTTPS address before starting n8n
6. Publish the workflow — Telegram's live delivery requires a running, published listener, not just a saved one

## What I'd build next

- Expense categories and per-category totals ("how much did I spend on food this month?")
- Due dates for tasks, with the AI extracting natural dates like "tomorrow" or "Friday"
- Deploying this to a real server, so it doesn't depend on a laptop staying on and an ngrok tunnel staying open
