# Inbound Email Triage — n8n + AI (classify → draft → human approval → reply)

A working workflow for the problem every small business has: **inquiries arrive faster than
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
reply sent (SMTP, Re: preserved)  →  logged to Sheets/Airtable
```

> **Threading caveat, stated upfront:** the SMTP send node cannot set `In-Reply-To` /
> `References`, so replies arrive as new conversations. Details and workarounds in
> [What is actually implemented vs merely recommended](#what-is-actually-implemented-vs-merely-recommended).


## Why the approval gate is not optional

Automated replies to real customers are how businesses lose customers. The **draft** is automated;
the **send** is a human decision. Categories you trust can later be switched to full auto; low
confidence messages always land in a human queue.

## What is in here

| File | What it is |
|---|---|
| [`workflow_email_triage.json`](workflow_email_triage.json) | Import-ready n8n workflow — 14 functional nodes plus 7 sticky notes explaining each step. Credential fields are placeholders — see below. |
| [`prompts_en.md`](prompts_en.md) | The classification + drafting prompts, including the JSON contract. |
| [`AUDIT-SAMPLE-workflow-reliability.md`](AUDIT-SAMPLE-workflow-reliability.md) | A real reliability audit run against the workflow above — 8 findings, including one that contradicts a claim this README used to make. |
| [`LICENSE`](LICENSE) | MIT — use it, adapt it, ship it. |

## Import (3 minutes)

1. n8n → **Workflows → Import from File** → `workflow_email_triage.json`
2. Create two credentials and re-select them on the nodes (IDs in the file are placeholders):
   - **IMAP** (read) — Gmail: enable IMAP + create an app password
   - **SMTP** (send) — same mailbox
3. Set the LLM node: base URL + API key (any OpenAI-compatible endpoint).
4. Replace the Sheets/Airtable node with your own sheet (or delete it).
5. Run it against 5 of your own emails before letting it near a shared inbox.

## What is actually implemented vs merely recommended

This distinction matters, so it is explicit. A previous version of this README described the
following as done. Some of it is not, and one item is not possible with the node used here.

| Claim | Reality in this export |
|---|---|
| Strict JSON output from the model | **Implemented.** Schema-constrained output, routing is a switch. |
| Low-confidence → human queue | **Implemented.** `confidence < 0.6` or `category = complaint` forces review. |
| Idempotency on `Message-ID` | **Partially.** IMAP unseen-set prevents re-fetch, but the webhook path has no `Message-ID` dedupe before side effects. |
| Reply stays in the same thread | **Not implemented, and not possible with this node.** See below. |

### Why replies do not thread (and what to use instead)

The `Send Email` (SMTP) node **cannot set `In-Reply-To` / `References` headers**. This is a
documented limitation, not a configuration mistake — see the "Limitations" section of the
n8n Send Email docs. Consequence: each automated reply arrives as a **new conversation** in the
customer's mailbox.

Workarounds, in order of preference:

1. **Gmail node → "Reply to a message"** operation. Threads correctly, but ties you to Gmail.
2. A **custom node** that accepts arbitrary headers.
3. Otherwise: accept thread-splitting, and keep the `Re:` subject so humans can still follow it.

This repo currently sends via SMTP with `Re:` preserved — correct for a demo, not enough for
production email. Know which one you are buying.


## Honest boundaries (what this repo does *not* do)

- No OCR / scanned-document field extraction: tested on text bodies, not on image-only PDFs.
- No telephony, no multi-tenant SaaS, no CRM write-back beyond a sheet/markdown log.
- **WhatsApp is deliberately out of scope**: unofficial libraries get numbers banned, and Meta's
  official Cloud API needs a verified business account plus pre-approved message templates.
- Outlook/Microsoft 365 works with the same IMAP/SMTP shape, but I have only run this end-to-end on Gmail.

## Status

Used daily on my own inbox. Classification is reliable; drafts are copy-paste-with-light-edits and are
meant to be reviewed. If you want this adapted to your categories and tone, that is the paid part.

## Work with me

- **Workflow reliability audit — USD 45, 48 hours.** You send the export; I send back a written
  failure map: the three most likely failure points, what you will see when each one fails, and the
  smallest fix. No instance access, no credentials.
- **Implementing the fixes** — USD 120–260, quoted before starting.
- **Monthly watch and fixes** — USD 99/month, 30 minutes of my time included.

A sample audit (of this very workflow) is in [`AUDIT-SAMPLE-workflow-reliability.md`](AUDIT-SAMPLE-workflow-reliability.md).
It reports eight findings against this repo, including one that contradicts a claim this README used
to make. That is the standard the paid work is held to.

Reach me at **abd315222@gmail.com**, or reply on the n8n community forum as **ck_lu**.

