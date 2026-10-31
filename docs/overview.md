# Meraki WhatsApp Automation: System Overview

**Scope:** The four connected systems behind the WhatsApp task bot and reporting pipeline: the WhatsApp Bot, the Cumulative Sales Report generator, the Missing Names Resolver, and the Management Tracker chatbot.

The exported workflow JSON files are in [`../workflows`](../workflows). They carry node logic and configuration only. Credentials (WhatsApp API keys, SMTP login, ClickUp API key) are stored separately in n8n and are **not** in the files; emails, paths and keys have been replaced with placeholders. To run any of these, reconnect your own credentials and update the placeholders.

---

## 1. How the pieces fit together

**The WhatsApp bot** is the hub. It's the single entry point for incoming WhatsApp messages. Depending on what the user asks for or where they are in a conversation, it:
- Parses intent and creates/updates tasks in ClickUp (the core "task bot" behavior)
- Tracks lightweight conversation state so multi-step replies (like "yes/no" or filling in a due date) make sense
- Looks up the sender's registered contact record and, if they're asking for the cumulative sales report, finds the latest generated report file and delivers it, both as a WhatsApp document and by email
- Sends a generic acknowledgment for anything it doesn't have a specific flow for

**Cumulative Sales Report** generates the report file on its own schedule; it's the producer. It pulls sales data and writes a dated Excel report to a shared folder. The bot is the consumer: when someone asks for "the report," it finds the newest file in that folder and sends it.

**Missing Names Resolver** is the data-quality safety valve. When the report run finds customer names that don't match the cleaning map, or customers with no account manager, it flags them instead of failing silently. The Resolver is a small set of web endpoints (a resolver page, a "get current state" endpoint and an "action" endpoint) that lets someone reconcile those items by hand.

**Management Tracker (chatbot)** is a separate, management-facing conversational flow aimed at tracking and summarizing status for leadership. It's built as an AI-agent flow rather than the rule-based intent parsing in the main bot.

In short: the bot is the always-on front door; the Sales Report workflow feeds it data on a schedule; the Names Resolver patches data gaps; and the Management Tracker is a parallel, higher-level view for leadership.

---

## 2. Status

| Workflow | Status | Notes |
|---|---|---|
| WhatsApp bot | **Live** | Core bot in daily use. |
| Cumulative Sales Report | **Live** | Scheduled report generation is working. |
| Missing Names Resolver | **Live** | Low-traffic utility, used as needed. |
| Management Tracker Chatbot | **In progress** | Several nodes are marked "not configured yet." Not production-ready. |

---

## 3. Bug found: unreliable report delivery

Users reported that the cumulative report "takes forever to come through" or sometimes never arrives.

The report-delivery branch runs two steps in parallel after a report request is recognized: sending the report as a WhatsApp document, and emailing a copy. The email step had a broken recipient expression. It referenced `.item.json.email` instead of `.first().json.email`, which resolved to empty under certain item-linking conditions and caused a "No recipients defined" error.

The bigger issue was that n8n's default "On Error" behavior is **Stop Workflow**, which aborts the whole execution on any unhandled node error, including sibling branches that are unrelated. So when the email step failed, it also killed the WhatsApp document delivery. It wasn't slow; it was failing silently downstream of a parallel error.

**Fix (applied in [`whatsapp-report-bot.json`](../workflows/whatsapp-report-bot.json)):**
1. Corrected the recipient expression to `{{ $('Parse & Decide').first().json.email }}`
2. Changed the email step's On Error setting from Stop Workflow to Continue, so an email failure no longer takes down WhatsApp delivery

---

## 4. Notes and next steps

- The **contact registry** (phone number to name/email) is a flat JSON config file on the host machine, not a database. It should move to something more robust as usage grows.
- The **Management Tracker** uses an AI-agent pattern, which differs from the rule-based parsing in the main bot. If both are maintained, it's worth standardizing on one approach.
- All workflows run on a single self-hosted n8n instance on a Windows machine, with no redundancy or failover.
