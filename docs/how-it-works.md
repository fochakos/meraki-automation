# How Each Workflow Works

## 1. WhatsApp bot (core)

- Entry point for every incoming WhatsApp message, via a webhook.
- Parses the message to figure out intent (new task, update, reply to a prompt, report request, etc.).
- Creates or updates tasks in ClickUp based on that intent, assigning them to team members.
- Tracks lightweight session/conversation state so multi-step exchanges (like confirming a due date) stay coherent across messages.
- On a report request: looks up the sender's phone number in a stored contact list, finds their email, and checks a shared folder for the latest generated report file.
- If found, delivers the report two ways in parallel: as a WhatsApp document attachment and as an email attachment.
- Sends a generic "thanks, we'll get back to you" acknowledgment for anything outside these flows.

## 2. Cumulative Sales Report (report generator)

- Runs on its own schedule, independent of any WhatsApp message.
- Pulls sales data from source systems.
- Builds a formatted Excel report from that data.
- Saves the finished file into a shared folder with a dated filename.
- Has no direct connection to WhatsApp. It only produces the file; the bot is what hands it to people.

## 3. Missing Names Resolver (data-gap tool)

- Exists because the report can only be accurate when every customer name is matched to a known record and account manager.
- Exposes a small resolver web page plus two supporting endpoints: one to fetch the current list of unmatched items, one to submit a fix.
- Used manually (not automatically triggered) to match an unrecognized name to the right record, add it as new, or skip it.
- Closes gaps that would otherwise leave items unresolved in the next report run.

## 4. Management Tracker Chatbot (in progress)

- A separate, management-facing flow that gives leadership a tracking/status view, distinct from the task-assignment bot.
- Built around an AI-agent node rather than the simpler rule-based parsing the main bot uses.
- Several nodes are explicitly marked as not configured yet.
- Not production-ready; treat it as a work in progress.
