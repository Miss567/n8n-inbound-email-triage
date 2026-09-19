# Inbound Email Triage — n8n + AI (classify → draft → human approval → reply in-thread)

A working, tested workflow for the problem every small business has: **inquiries arrive faster than
anyone can answer them, so leads go cold.** This repo is the pipeline I run on a real mailbox.

```
incoming email (IMAP)
      │
      ▼
normalise (subject / from / thread id / body)
      │
      ▼
AI classification  ── strict JSON: {category, urgency, reference_no, needs_human}
      │
      ▼
AI draft reply  (your tone)
      │
      ▼
HUMAN APPROVAL  ← nothing is ever sent without a click
      │
      ▼
reply sent in the same thread (SMTP, Re: preserved)  →  logged to Sheets/Airtable
```

## Why the approval gate is not optional

Automated replies to real customers are how businesses lose customers. The **draft** is automated;
the **send** is a human decision. Categories you trust can later be switched to full auto; low
confidence messages always land in a human queue.

## What is in here

| Path | What it is |
|---|---|
| `n8n/workflow_email_triage.json` | Import-ready n8n workflow (14 nodes). Credential fields are placeholders — see below. |
| `prompts/prompts_en.md` | The classification + drafting prompts, including the JSON contract. |

## Import (3 minutes)

1. n8n → **Workflows → Import from File** → `n8n/workflow_email_triage.json`
2. Create two credentials and re-select them on the nodes (IDs in the file are placeholders):
   - **IMAP** (read) — Gmail: enable IMAP + create an app password
   - **SMTP** (send) — same mailbox
3. Set the LLM node: base URL + API key (any OpenAI-compatible endpoint).
4. Replace the Sheets/Airtable node with your own sheet (or delete it).
5. Run it against 5 of your own emails before letting it near a shared inbox.

## Design decisions worth copying

- **Strict JSON output from the model.** Free-form text makes downstream routing unreliable. With a
  schema, routing is a switch node — not another model call — and every decision is auditable.
- **Thread preservation on send.** Set `In-Reply-To` / `References` and keep the `Re:` subject, or
  your "automated" reply thread-splits in the customer's mailbox.
- **Idempotency.** Key each message on its `Message-ID` and store it before any side effect, so a
  retry cannot send twice.
- **Low-confidence → human queue.** A false "urgent" costs a click; a false "not urgent" costs a customer.

## Honest boundaries (what this repo does *not* do)

- No OCR / scanned-document field extraction: tested on text bodies, not on image-only PDFs.
- No telephony, no multi-tenant SaaS, no CRM write-back beyond a sheet/markdown log.
- **WhatsApp is deliberately out of scope**: unofficial libraries get numbers banned, and Meta's
  official Cloud API needs a verified business account plus pre-approved message templates.
- Outlook/Microsoft 365 works with the same IMAP/SMTP shape, but I have only run this end-to-end on Gmail.

## Status

Used daily on my own inbox. Classification is reliable; drafts are copy-paste-with-light-edits and are
meant to be reviewed. If you want this adapted to your categories and tone, that is the paid part.
