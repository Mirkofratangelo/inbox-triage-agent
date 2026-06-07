# Setup — full walkthrough

This is the long version of the setup section in
[`../README.md`](../README.md), written for people who haven't used
n8n or Google Cloud before. If you have, you can probably skip
straight to step 5.

End to end this takes about **20–30 minutes** the first time. After
that, importing the workflow again into a new n8n is a 2-minute job.

> **A note before you start.** Everything in this guide is free at
> the volumes a small business runs at. Anthropic gives you free
> credits when you sign up, Gmail and Google Sheets are free, n8n
> is free to run locally. The only thing it will eventually cost
> you is Anthropic API tokens — roughly **$0.50–$3 a month** on a
> typical small inbox, after the pre-filter does its job.

---

## 1. Install n8n

n8n is the workflow engine that runs the triage. You can install it
in many ways; the simplest is to run it directly from your terminal.

```bash
npx n8n
```

The first run downloads n8n (~5 minutes), then opens
`http://localhost:5678` in your browser and asks you to create a
local account. That account stays on your machine — no cloud, no
sign-up.

If you'd rather have a hosted instance, n8n Cloud, Railway, and
Render all work — the workflow.json is the same.

## 2. Get an Anthropic API key

The triage and the draft replies are written by Claude (Anthropic's
model). You need one API key, and the same key is used by both
nodes.

1. Go to [console.anthropic.com](https://console.anthropic.com) and
   sign up. New accounts get free credits.
2. In the left sidebar, click **API Keys → Create Key**.
3. Copy the key. **Save it somewhere safe — Anthropic only shows
   it once.**

If you ever lose it, you can revoke the old one and create a new
one. No data is lost.

## 3. Create the Google Sheet

The workflow writes its output here. You need one spreadsheet with
**two tabs**, named **exactly** as shown below (the workflow looks
them up by name — a typo will silently break it).

1. Open [sheets.google.com](https://sheets.google.com) and create
   a blank sheet. Name it something like *"Inbox Triage"*.
2. Rename the first tab to **`Bozze da rivedere`** and add this
   header row in row 1:

   | A    | B    | C       | D         | E     |
   | ---- | ---- | ------- | --------- | ----- |
   | data | from | subject | messaggio | bozza |

3. Add a second tab named **`Richiede te`** with this header row:

   | A    | B    | C       | D         | E      |
   | ---- | ---- | ------- | --------- | ------ |
   | data | from | subject | messaggio | motivo |

4. Copy the spreadsheet's **ID** — the long string in the URL
   between `/d/` and `/edit`:

   ```
   https://docs.google.com/spreadsheets/d/THIS_IS_THE_ID/edit
   ```

   You'll paste it into the workflow in step 6.

## 4. Set up Google OAuth (Gmail + Sheets)

Both the Gmail node and the Sheets nodes need a Google OAuth
credential to read your mail and write to your sheet. You can use
**one OAuth client for both** — that's the simpler path.

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
   and create a new project (top-left dropdown, **New Project**).
2. Inside the project, open **APIs & Services → Library** and
   enable these two APIs:
   - **Gmail API**
   - **Google Sheets API**
3. Open **APIs & Services → OAuth consent screen**:
   - User type: **External**
   - App name: anything, e.g. *Inbox Triage*
   - Add your own Gmail address as a test user
4. Open **APIs & Services → Credentials → Create Credentials →
   OAuth client ID**:
   - Application type: **Web application**
   - Authorized redirect URI: get this from n8n. In n8n, click
     any Gmail node, choose **Create New Credential**, and copy
     the **OAuth Redirect URL** shown at the top of the credential
     form. Paste it back into Google Cloud.
5. Google gives you a **Client ID** and **Client Secret**. Keep
   them open in another tab — you'll paste them into n8n next.

> If this feels like a lot of steps: it's a one-time Google
> setup. Once the OAuth client exists, every n8n workflow you
> ever build can reuse it.

## 5. Import the workflow

1. Download `workflow.json` from this repo.
2. In n8n, top-right **⋯ menu → Import from File** and pick the
   downloaded file. You'll land on the canvas with 13 nodes.

## 6. Wire up the credentials

The imported workflow has empty credential slots on five nodes.
Click each one and fill it in.

### Anthropic credentials (two nodes)

| Node                  | Credential type         |
| --------------------- | ----------------------- |
| `Anthropic - classify` | Anthropic               |
| `Anthropic - draft`    | Anthropic               |

Click the first node, **Create New Credential**, paste your API
key from step 2, save. On the second node, just pick the same
credential from the dropdown — don't create a new one.

### Gmail credential (one node)

- Node: `Gmail`
- Credential type: **Gmail OAuth2**

Click **Create New Credential**, paste the Client ID and Client
Secret from step 4, click **Sign in with Google**, and approve the
permissions in the popup. n8n stores the OAuth refresh token.

### Google Sheets credential (two nodes)

| Node                          | Credential type            |
| ----------------------------- | -------------------------- |
| `Sheets - Bozze da rivedere`  | Google Sheets OAuth2       |
| `Sheets - Richiede te`        | Google Sheets OAuth2       |

Same pattern as Gmail: paste the same Client ID / Client Secret
once into the first Sheets node, **Sign in with Google**, then on
the second node just pick the existing credential from the
dropdown.

### Spreadsheet ID

In both Sheets nodes, find the **Document** field. It still says
`YOUR_SPREADSHEET_ID`. Replace it with the real ID you copied in
step 3.

The **Sheet** field on each node already points to the correct
tab name (`Bozze da rivedere` on one, `Richiede te` on the other).
Don't change those.

## 7. First run — keep it small

Before you hit Execute, open the **Gmail** node and look at the
**Limit** field. The workflow ships with **Limit: 50**, but on
your first run **set it to 10 or 20**. This:

- caps how many emails get classified (cheap),
- lets you see how the pre-filter regex behaves on YOUR inbox,
- and lets you read every row in the Sheet to spot
  misclassifications.

Click **Execute Workflow** in the top-right.

After the run, check:

- the green checkmarks on each node — they tell you how many items
  passed through. The number on **Pre-filter spam** is how many
  emails Claude actually got to see; the difference is how many
  the regex saved you from paying for.
- the two tabs of your Sheet — new rows should appear in
  **Bozze da rivedere** and **Richiede te**.

## 8. Tune the pre-filter (optional, but worth 10 minutes)

The **Pre-filter spam** node has two regex patterns:

- `NOISE_SENDER_RE` — addresses that should never see the LLM
  (`noreply@`, `notifications@`, etc.)
- `NOISE_SUBJECT_RE` — subjects that signal newsletter / promo
  blasts.

After your first run, look at:

- emails that **shouldn't have reached Claude** but did (e.g. a
  Shopify order notification that ends up in `richiede_te`). Add
  the sender pattern to `NOISE_SENDER_RE`.
- emails that **got dropped but shouldn't have** (a real customer
  with a `notifications` in their domain). Tighten the regex.

This tuning is the cheapest performance lever in the workflow.
Every email you catch here is an email you don't pay Claude to
read.

## 9. Put it on a schedule

Once you trust the output:

1. Delete the **Manual Trigger** node.
2. Add a **Schedule Trigger** node in its place. Every 15–60
   minutes works well.
3. Wire it to the **Gmail** node and **Activate** the workflow
   (toggle top-right).

It will now run quietly in the background. You only ever look at
the Google Sheet.

---

## Troubleshooting

**Claude says the JSON is invalid (Code node throws).** The
classifier system prompt is strict, but Claude occasionally wraps
its reply in backticks or adds a sentence. The Parse nodes catch
this with a clear error message that includes the bad output.
Open the failed execution, copy the output Claude returned, and
either tune the system prompt in the Agent node or open an issue
in this repo — I'm collecting these cases for the next version.

**Gmail returns 0 messages.** The `Filters` field on the Gmail
node is empty by default — it pulls the latest messages from your
INBOX. If you have inbox filters in Gmail that auto-archive most
mail, the workflow will see less than you expect. Use the `q`
search filter (e.g. `newer_than:1d in:inbox`) to be explicit.

**The Sheet shows partial rows.** Check that both tab names match
exactly — including capitalization and the space in "Richiede te"
or the spaces in "Bozze da rivedere". The Sheets nodes look them
up by name.

If you hit something this guide doesn't cover, open an issue. The
fastest way to make the docs better is for someone to tell me
where they got stuck.
