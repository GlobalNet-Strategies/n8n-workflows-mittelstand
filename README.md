# n8n-workflows-mittelstand

Generic, production-style n8n workflow examples for mid-sized companies:
request routing, invoice approval and employee onboarding — each with an
explicit error path and a manual review step wherever data is uncertain.

These are **generic examples, no client data**: no credentials, no real
accounts, domains or names. Every `YOUR_*` placeholder must be replaced
before use. Maintained by [GlobalNet Strategies LLC](https://globalnet-strategies.com),
which plans, builds and runs n8n workflows for mid-sized companies.

## Workflows

| File | What it does | Trigger | Key nodes |
|---|---|---|---|
| `workflow-1-route-quote-requests.json` | Collects quote requests from two shared IMAP mailboxes, classifies them by product group, checks for duplicates (same sender and subject within one trigger run; each mailbox trigger runs separately), upserts a HubSpot contact and notifies the owner — or routes to a manual review inbox | Email Trigger (IMAP) | IMAP, Set, Code, If, HubSpot, Send Email |
| `workflow-2-route-incoming-invoices.json` | Reads invoices from one central mailbox, extracts invoice number, vendor, amount and cost center for every mail of the run (its duplicate check only compares invoice numbers within one trigger run, so it does not catch an invoice re-sent in a later run), and dispatches each invoice to the per-invoice sub-workflow without waiting for it | Email Trigger (IMAP) | IMAP, Set, Code, SplitInBatches, HTTP Request |
| `workflow-2b-process-single-invoice.json` | Processes one invoice in its own execution: flags uncertain values for review and asks the approver to approve or decline directly in the email (Send and Wait for Response). No response triggers a reminder and a second request; a second timeout escalates to the fallback approver. Requires n8n 2.30.0 or later (response detection reads `data.respondedAt`) | Webhook | Webhook, Code, If, Send Email (Send and Wait for Response) |
| `workflow-3-onboarding-tasks.json` | Reads an HR-approved hire list from Google Sheets once per morning, creates onboarding tasks (management, department, IT, equipment) with due dates derived from the start date, and emails only the tasks due that day (Europe/Berlin) | Schedule (cron) | Schedule Trigger, Google Sheets, Set, Code, If, Send Email |

## Import

1. In n8n, open **Workflows → ⋯ → Import from File** and pick a JSON file
   (or paste the file content into the canvas). Workflow 2 comes in two
   files — import both.
2. Replace every `YOUR_*` placeholder and the `workflow@example.com` sender
   address: owners, inboxes, sender address and day counts in each
   `… Config` Set node; mailbox folders in the IMAP nodes; sheet ID and
   sheet name in the Google Sheets node; the admin and sender addresses in
   `Alert Admin on Error`; and the webhook path that connects workflow 2 and
   workflow 2b — set the same `YOUR_INVOICE_PROCESSOR_WEBHOOK_PATH` in
   workflow 2b's webhook and in the URL of workflow 2's
   `Dispatch per Invoice` node (with your n8n host).
3. Connect the credentials listed below (types only — nothing is embedded).
   Workflow 2 and 2b share one additional **Header Auth** credential that
   authenticates the call between them: create a Header Auth credential with
   any header name and a secret value, then select it on workflow 2b's
   `Invoice Received` webhook and on workflow 2's `Dispatch per Invoice` node
   — the same header name and value on both ends. Requests without the
   matching header are rejected before the sub-workflow runs, so only the
   router (holding the secret) can trigger approval mails.
4. Activate the workflows. For workflow 2 this matters twice over: the
   webhook of workflow 2b only exists while workflow 2b is active,
   and the IMAP trigger only polls while workflow 2 is active.

## Required credentials (types only, no values included)

| Workflow | Credential types |
|---|---|
| 1 — quote requests | IMAP (for both mailbox triggers), SMTP (send email), HubSpot Service Key (preset in this workflow; named "App Token" before n8n 2.18.0) or HubSpot OAuth2 |
| 2 — incoming invoices | IMAP (workflow 2), SMTP (both files), Header Auth (one shared credential, selected on both files) |
| 3 — onboarding tasks | Google Sheets OAuth2, SMTP |

## Design notes

- **Error paths:** every node with an external dependency (IMAP, Google
  Sheets, HubSpot, SMTP) uses *On Error: Continue (using error output)* —
  except `Alert Admin on Error` itself, which continues on its regular output.
  All error outputs meet in a `Log Error` node that mails the admin. The admin
  address is set directly in the `Alert Admin on Error` node, not in the
  config Set node (in workflow 3 the Google Sheets read runs before it).
  Workflow 2 splits this across its two files: the router handles its IMAP
  trigger and the dispatch node, the per-invoice sub-workflow handles its SMTP
  nodes — each file has its own `Log Error` / `Alert Admin on Error` pair.
  Trigger failures (for example the IMAP connection) never reach an error
  output: n8n runs the workflow's Error Workflow instead. To be alerted about
  them, register a separate workflow with an
  [Error Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.errortrigger/)
  under *Settings → Error workflow*.
- **Due-date gating (workflow 3):** a task is only sent on the day its due
  date falls on; "today" is computed in Europe/Berlin inside the Code node.
  If the start date moves, the task is simply sent on the new due date —
  no task repeats daily and nothing fires on the old date. Only the HR mail
  about incomplete approved rows repeats every morning until the row is
  fixed. The workflow's timezone setting is Europe/Berlin, so the 06:00
  schedule and "today" use the same calendar day.
- **Manual review:** data that cannot be classified with confidence
  (missing sender, unclear invoice fields, incomplete hire rows, duplicates)
  is routed to a review inbox with the reason — never guessed into a CRM,
  approval chain or task list. Workflow 2 keeps this promise per invoice:
  every invoice mail of a trigger run is dispatched to its own execution of
  workflow 2b, so approval, review and log steps always work with that
  invoice's data — including the backlog runs at first activation (all
  unread mails) and after trigger downtime (n8n restart, IMAP reconnect,
  deactivation). The duplicate check stays honest: it compares invoice
  numbers within one run only, so a duplicate inside a run goes to review,
  while an invoice re-sent in a later run is not caught.
- **Config:** owners, inboxes, sender address and day counts live in each
  workflow's `… Config` Set node (in workflow 3 it follows the Google Sheets
  read); edit there, not in the Code nodes. In workflow 2 the extraction
  copies every config value the invoice needs downstream into its item (the
  cost-center map is resolved to the approver first), and the sub-workflow
  reads its day counts from that item. Node-specific values stay in their
  node: mailbox folders (IMAP), sheet ID and name (Google Sheets), admin and
  sender address (`Alert Admin on Error`), and the sub-workflow reference
  (`Dispatch per Invoice`).
- Sticky notes on the canvas document every section.

## License

MIT — see [LICENSE](LICENSE).
