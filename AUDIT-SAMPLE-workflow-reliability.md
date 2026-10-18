# Workflow Reliability Audit — sample

**Subject:** `Triage inbound email with IMAP, custom AI HTTP APIs and SMTP` (public template 19734)
**Scope:** 15 functional nodes + 6 sticky notes. Read-only review of the exported workflow JSON.
**Method:** simulate each failure mode against the node configuration. No live system was touched.

This is what a $45 audit deliverable looks like. Yours would cover your workflow, not this one.

---

## Verdict

The happy path is well built: normalisation, strict-JSON classification, a human-approval
branch, and a log write on both branches. The problem is not the happy path. It is what
happens on the unhappy one — and in this export, **nothing in the workflow reports that it
failed**. Below are the findings in the order I would fix them.

---

## Findings

### F1 — An inquiry can disappear with no trace (high)

`IMAP: fetch unseen email` uses `postProcessAction: read`, so the message is marked read the
moment it is fetched. If classification, drafting, or the database write fails afterwards, the
message is already read, is never retried, and never appears in the log.

**How you would see it:** a customer says "I emailed you last Tuesday" and there is no row for
it. You cannot reproduce it, because the email is gone from the unseen set.

**Fix:** keep the message unread until the log write succeeds (fetch with `postProcessAction:
nothing`, then mark read as the last step), or write a `received` row before any AI call.

---

### F2 — No failure ever reaches a human (high)

Every functional node has `retryOnFail`, `onError` and `continueOnFail` unset, and the workflow
has **no error workflow** in `settings`. A 30-second timeout on the classify or draft API ends
the execution silently.

**How you would see it:** you would not. You would find out from the missed lead.

**Fix:** set an error workflow that posts to Slack/email, and add retries (2–3, backoff) on the
two HTTP AI calls. Those are the nodes most likely to fail transiently.

---

### F3 — The "same thread" claim is not implemented, and cannot be with this node (high)

The step-6 note says replies go out with `In-Reply-To / References`, so they stay in the thread.
The `SMTP: reply in the same thread` node sets only `from`, `to`, `subject`, `text` and has an
empty `options` object — **no headers**.

More important than the missing parameter: n8n's own documentation lists this as a **limitation
of the Send Email node** — it cannot set `In-Reply-To` / `References`, so every reply arrives as
a new conversation. The node name and the sticky note describe a behaviour the node cannot
produce.

**How you would see it:** the customer's thread breaks; follow-up conversations fragment; the
promise in your handover note is false.

**Fix:** either use the **Gmail node's "Reply to a message"** operation (threads correctly, ties
you to Gmail), or a custom node that accepts arbitrary headers. If neither is acceptable, change
the claim — a documented limitation is fine, a false claim in the handover note is not.


---

### F4 — Empty drafts can be sent (high, only if auto-send is on)

`draft: inputItem.data?.draft ?? inputItem.draft ?? ''` falls back to an empty string, and the
same fallback sets `confidence` to `1` when the field is missing. A classifier that returns
valid JSON but no `confidence` is therefore treated as **fully confident**, and an empty draft
can pass the approval gate.

**Fix:** treat a missing `draft` or `confidence` as a review trigger, not as a default. Never
send an empty body.

---

### F5 — Webhook entry points are unauthenticated (medium)

Both webhook triggers accept any POST to their path with `responseMode: onReceived` and no
secret check. Anyone who learns the path can inject a message into the pipeline, which then
gets classified, drafted, and potentially sent.

**Fix:** require a shared secret header, or keep the webhook triggers inactive in production
and use IMAP only.

---

### F6 — No idempotency on the webhook path (medium)

Deduplication relies on IMAP's unseen set. The webhook path has no `messageId` uniqueness
check, so a retried webhook delivery is processed twice — and with auto-send enabled, replied
to twice.

**Fix:** dedupe on `messageId` in the log table before the classify step.

---

### F7 — Documentation does not match the workflow (low, but erodes trust)

The trigger node is named `Schedule Trigger: poll every 5 min`, but its rule is 15 minutes.
Small, but this is exactly the kind of mismatch that makes a client distrust the handover note.

**Fix:** rename or fix the interval.

---

### F8 — Customer content leaves your infrastructure without redaction (medium, compliance)

Message bodies (up to 20,000 characters) are posted to external AI endpoints. That is the
design, and it may be fine — but if the inbox contains personal data, you need to be able to
say what leaves and where it goes.

**Fix:** decide explicitly (log a one-line data-flow note), or redact obvious identifiers
before the call.

---

## What I would do first, in order

1. F2 — error workflow + retries. Without it, nothing else matters: you cannot see failures.
2. F1 — do not mark mail read before it is safely logged.
3. F3 — fix the threading headers (the promise on the sticky note is currently false).
4. F4 — never default confidence to 1; never send an empty draft.
5. F5/F6 — authenticate webhooks and dedupe on messageId.

Items 1–3 are roughly an hour of work and remove the three failures that cost the most.

---

## What this audit is not

It is not a penetration test, a performance test, or a review of the AI prompt quality. It
reads the workflow configuration and reasons about failure modes. If you want the fixes
implemented rather than listed, that is quoted separately before any work starts.

— Ck Lu · `abd315222@gmail.com`
