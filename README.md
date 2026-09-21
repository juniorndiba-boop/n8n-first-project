# Customer Inquiry Workflow (n8n)

An n8n automation that turns a website inquiry form into a logged lead, an AI-drafted reply waiting in Gmail, and an instant Telegram notification.

Built for a climate resilience and Nature-based Solutions (NbS) consultancy, but the structure works for any service business that receives inquiries through a form.

---

## What it does

When someone submits the inquiry form, the workflow:

1. Receives the submission via webhook
2. Extracts the name, email, phone number and inquiry text
3. Generates a personalised, professional reply using an LLM
4. Appends the lead's details to a Google Sheet
5. Creates a Gmail draft containing the generated reply
6. Sends a Telegram message so you know an inquiry has come in

The reply is created as a **draft, not an auto-send**. A human reviews and sends it. That is deliberate: it keeps a person in the loop on every client-facing message.

---

## Workflow structure

```
Webhook
   ↓
Edit Fields          (map form fields to named variables)
   ↓
Message a model      (generate the client reply)
   ↓
Append row in sheet  (log the lead)
   ↓
Create a draft       (Gmail draft with the reply)
   ↓
Send a text message  (Telegram alert)
```

### Nodes

| Node | Type | Purpose |
|---|---|---|
| Webhook | `n8n-nodes-base.webhook` | POST endpoint that receives the form submission |
| Edit Fields | `n8n-nodes-base.set` | Maps the raw payload into `Name`, `Email`, `Phone Number`, `Inquiry` |
| Message a model | `@n8n/n8n-nodes-langchain.openAi` | Generates the client-facing reply from a structured prompt |
| Append row in sheet | `n8n-nodes-base.googleSheets` | Logs the inquiry to a Google Sheet |
| Create a draft | `n8n-nodes-base.gmail` | Saves the reply as a Gmail draft for review |
| Send a text message | `n8n-nodes-base.telegram` | Notifies you that a new inquiry has arrived |

---

## Requirements

- An n8n instance (cloud or self-hosted)
- An LLM API credential compatible with the OpenAI node (this build uses a `qwen-plus` model)
- A Google account with Sheets and Gmail access
- A Telegram bot token and your chat ID
- A form provider that can send a webhook (this build uses Tally)

---

## Setup

### 1. Import the workflow

In n8n: **Workflows → Import from File** and select `Customer_Inquiry_Workflow.json`.

### 2. Add your credentials

The imported workflow has empty credential slots. Open each node and connect your own:

- **Message a model** → your LLM API credential
- **Append row in sheet** → Google Sheets OAuth2
- **Create a draft** → Gmail OAuth2
- **Send a text message** → Telegram API

### 3. Point the form at the webhook

Copy the production webhook URL from the Webhook node and add it to your form provider's webhook settings.

### 4. Check the field mapping

The Edit Fields node reads fields by position:

```
$json.body.data.fields[0].value   → Name
$json.body.data.fields[1].value   → Email
$json.body.data.fields[2].value   → Phone Number
$json.body.data.fields[3].value   → Inquiry
```

If your form has a different field order, update the indexes. Run the workflow once in test mode, inspect the webhook output, and map from what you actually receive.

### 5. Connect your sheet

Select your own spreadsheet and tab in the Google Sheets node. The sheet needs these column headers:

```
Name | Email | Phone Number | Inquiry
```

### 6. Set the Gmail recipient

Add the recipient address in the Gmail node so the draft is addressed to the client.

### 7. Set your Telegram chat ID

Replace the chat ID in the Telegram node with your own. You can get it by messaging your bot and calling `https://api.telegram.org/bot<TOKEN>/getUpdates`.

---

## The prompt

The reply quality lives almost entirely in the system prompt inside the **Message a model** node. It sets out:

- the consultancy's scope and service areas
- the client's name, email, phone and inquiry as variables
- a target length of 80 to 150 words
- a rule to address the client by first name and answer directly
- a rule to prioritise Nature-based Solutions only where genuinely relevant
- a ban on filler openers, sales language, invented details, price quotes and guarantees
- a requirement to end with one clear next step
- a rule never to mention AI, automation or the tooling behind the reply

Edit this prompt to match your own business. It is the main thing you will tune.

---

## Customising it

Some straightforward changes:

- **Auto-send instead of draft** — switch the Gmail node's resource from `draft` to `message`. Only do this once you trust the output.
- **Different notification channel** — swap the Telegram node for Slack, WhatsApp or email.
- **Route by inquiry type** — add a Switch node after Edit Fields to handle different inquiry categories differently.
- **Add a CRM** — replace or supplement the Google Sheets node with Airtable, Notion or HubSpot.
- **Add validation** — insert an IF node to drop submissions with a missing email or empty inquiry.

---

## Notes and limitations

- Field mapping is positional, so reordering the form silently breaks it. Named-field mapping is more robust if your form provider supports it.
- There is no retry or error branch. If the LLM call fails, the run stops and nothing downstream executes.
- The Gmail node is set to HTML but the generated reply is plain text, so line breaks may not render as expected. Either convert newlines to `<br>` or switch the email type to plain text.
- The workflow assumes one submission per webhook call.

---

## Repository contents

```
.
├── Customer_Inquiry_Workflow.json
└── README.md
```

---

## Before you push

The exported JSON includes credential IDs, a Google Sheet ID, a Telegram chat ID and webhook IDs. These are references rather than secrets (no tokens or API keys are exported), but they still identify your accounts. Replace them with placeholders before making the repository public.

---

## License

MIT
