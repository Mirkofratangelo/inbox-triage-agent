# Inbox Triage Agent

The light triage layer that surfaces the few messages that need you
and filters the rest — without another subscription.

A free, self-hosted [n8n](https://n8n.io) workflow that reads your
inbox, classifies each email with [Claude](https://www.anthropic.com),
pre-writes draft replies for the routine FAQs, and writes everything
into a Google Sheet you actually read. The AI never sends anything on
its own. You stay in charge of every reply.

---

## The problem

If you run a small store or a one-person business, your inbox is
mostly the same five questions on repeat — *where is my order*,
*can I return this*, *do you ship to X*. Buried in there are the
three or four messages that actually need a decision: a complaint,
a refund request, a wholesale inquiry. They get lost in the noise,
or they sit unanswered for a day.

You don't need another helpdesk subscription. You need a quieter
inbox and a head start on the boring replies.

## What it does

1. **Classifies every email** into one of three buckets — `rumore`
   (noise: newsletters, no-reply blasts), `richiede_te` (needs you:
   refunds, decisions, anything ambiguous), or `auto_bozza` (routine
   FAQ with a standard answer).
2. **Pre-writes a draft reply** for the FAQ bucket and drops it in a
   Google Sheet next to the original message. You read it, edit if
   you want, and send. Or you don't.
3. **Surfaces only what needs you** in a separate tab. Newsletters
   and promo blasts get filtered before they ever touch the AI —
   that part runs in plain JavaScript, costs nothing, and is the
   first thing you'll want to tune.

**Human-in-the-loop is the whole point.** The workflow never
auto-replies on your behalf. It writes drafts you approve. The
classifier is explicitly told that when in doubt, it must escalate
to you — that rule is hard-coded in the system prompt.

**Cheap to run.** The pre-filter throws out obvious noise *before*
calling Claude, so you only pay for emails that might actually need
a reply. On a typical small-business inbox of ~500 messages a day,
expect roughly **$0.50–$3 a month** in Anthropic credits after the
pre-filter does its job.

## How it works

```
Gmail ──► Pre-filter (regex)
            │
            ▼
        Classify with Claude
            │
            ▼
     Route by category ──► rumore       (dropped)
            │              richiede_te ──► Google Sheet, tab "Richiede te"
            │
            └─► auto_bozza ──► Generate draft reply with Claude
                                       │
                                       ▼
                              Google Sheet, tab "Bozze da rivedere"
```

13 nodes, no external state, no database. The whole thing imports
into a fresh n8n in about a minute.

![n8n workflow canvas](docs/img/n8n-canvas.png)

A finished run looks like this — the Sheet holds all the rows, and
you work through them like a to-do list:

![Example digest in Google Sheets](docs/img/digest-esempio.png)

A draft reply close-up — you can see the original `messaggio` and
the AI-written `bozza` side by side:

![Example draft reply](docs/img/bozza-esempio.png)

## Setup (5 minutes if you've used n8n, 20 if you haven't)

You'll need:

- A running [n8n](https://docs.n8n.io/hosting/installation/) — the
  simplest way is `npx n8n` in a terminal.
- An [Anthropic API key](https://console.anthropic.com).
- A Gmail account (it works with the free tier).
- A Google Sheet with two empty tabs named exactly:
  - **`Bozze da rivedere`** — header row: `data | from | subject | messaggio | bozza`
  - **`Richiede te`** — header row: `data | from | subject | messaggio | motivo`

Quick steps:

1. In n8n, **Workflows → ⋯ → Import from File** and pick
   `workflow.json` from this repo.
2. Click each node that says "(no credentials)" and pick (or create)
   the right credential — Anthropic on the two model nodes, Gmail
   on the Gmail node, Google Sheets on both Sheets nodes.
3. In the two Sheets nodes, replace `YOUR_SPREADSHEET_ID` (the
   **Document** field) with your sheet's ID — the long string in
   the URL between `/d/` and `/edit`.
4. **Set the Gmail node's `Limit` to 10 or 20 for the first run.**
   This caps Anthropic usage while you're still tuning the
   pre-filter regex. Bump it up later.
5. Click **Execute Workflow** and check both tabs of your sheet.

Step-by-step instructions for non-technical setup
(OAuth screens, where each value goes, screenshots) live in
[`docs/setup.md`](docs/setup.md).

## What it does NOT do (on purpose)

This is the free version. It is deliberately small.

- **One inbox only.** No multi-mailbox aggregation. If you need
  several inboxes triaged into one place, that's a different shape
  of tool.
- **Email only.** No Instagram DMs, no WhatsApp, no Shopify
  notifications. Pure SMTP / IMAP / Gmail.
- **Three categories, fixed.** No custom labels, no per-product
  routing, no priority scoring.
- **Never sends a reply.** The "auto" in `auto_bozza` is short for
  "auto-drafted", not "auto-sent". Every reply is approved by you.

The limits are the point. Most paid inbox tools fail because they
try to do everything and end up being another inbox you have to
manage. This one does one job — sort the pile so you can stop
opening the same emails twice.

## Not using Gmail?

Swap the **Gmail** node for n8n's **Email Trigger (IMAP)** node.
The downstream shape (`subject`, `from`, `text`) is the same and
nothing else needs changing. Tune the regex in the **Pre-filter
spam** node to match whatever no-reply patterns your provider
uses.

## A note on what's next

I'm building a fuller version with multi-inbox support and
categories you can define yourself. If you try this and it helps,
or if you try it and something obvious is missing — open an issue
or send me a note. Real feedback from real inboxes is what makes
the next version good.

## License

[MIT](LICENSE). Fork it, modify it, ship it.

Built on top of [n8n](https://n8n.io) (Apache 2.0 with Commons
Clause) and [Anthropic Claude](https://www.anthropic.com).

---

Built by [Mirko Fratangelo](https://github.com/Mirkofratangelo) for
people whose inbox is a job they didn't sign up for.
