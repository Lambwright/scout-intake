# INTAKE

Standalone companion app for SCOUT — creates a new NetSuite Customer (and one or
more linked Contacts) from a pasted RFQ email, for staff who don't have NetSuite
access. Single file, no build step, same visual system as SCOUT.

Works two ways:
- **Standalone**, for anyone with an Einbau ID login (e.g. the CR Manager) —
  `https://lambwright.github.io/scout-intake/`
- **As a popout from SCOUT** — opened via the "+ New Company" link next to the
  client field. Hands off the logged-in session automatically (no second login)
  and reports the newly created customer back so SCOUT can select it.

## Auth

Gated by the shared Einbau ID Worker (`auth.ben-a90.workers.dev`) — same login
system as SCOUT. See the `auth-worker` repo for how that's deployed.

Handoff protocol when opened as a SCOUT popout (both sides check
`event.origin` before acting on a message):

1. INTAKE posts `{source:'einbau-intake', type:'INTAKE_READY'}` to `window.opener`.
2. SCOUT replies `{source:'einbau-scout', type:'SCOUT_INIT', token, rfqEmailText}`.
3. INTAKE verifies the token against `/auth/verify`; on success it skips its own
   login screen and pre-fills the email textarea from `rfqEmailText` if empty.
4. On submit: `{type:'INTAKE_COMPANY_CREATED', customer:{id,name,procoreId}}` back
   to SCOUT, then the popout closes itself. On a fatal error:
   `{type:'INTAKE_ERROR', message}` instead.

Standalone sessions live in `localStorage` (`einbau_intake_token`); popout
sessions live in `sessionStorage` so they don't persist past the tab closing.

## Parsing

Calls `scout.ben-a90.workers.dev` (the existing Anthropic proxy SCOUT already
uses) directly — no dedicated backend. Prompt asks Claude for:

```json
{"company":{"name":"","street":"","city":"","province":"","postal":"","country":"","phone":"","website":""},
 "contacts":[{"fname":"","lname":"","title":"","email":"","phone":""}],
 "needs_attachment":false,"needs_attachment_reason":"","confidence_notes":""}
```

It's told to prioritize the signature block and flag `needs_attachment` when it
suspects a signature image it can't read from plain text — INTAKE then offers to
re-parse with an attached screenshot/PDF (sent as a vision content block, same
pattern SCOUT's Copilot already uses for file attachments).

**Anything populated from parsed email content is written via `.value`/
`.textContent`, never `innerHTML`** — the email is untrusted input from an
external sender, and the page holds a live session token while open.

## NetSuite writes

Via `netsuite.ben-a90.workers.dev` (the existing generic NetSuite REST proxy):

1. **Customer** — `POST /customer`: `companyname`, `isPerson:false`,
   `subsidiary:{id:'2'}` (Einbau Services Ltd. — confirm this matches your
   account before relying on it), `isinactive:false`, optional `phone`/`url`,
   and an `addressbook` entry if any address field was filled in.
2. **Contact(s)** — `POST /contact` per row, sequential, once the customer id is
   known: `firstname`, `lastname`, `title`, `email`, `phone`, `company:{id}`.

New-customer id is read from the Worker's `location` field (set when NetSuite
returns an empty 204 body) with a fallback to a SuiteQL re-query by exact name
if that's ever missing — this fallback path hasn't been exercised against real
production NetSuite yet, worth watching on first real use.

A non-blocking duplicate-name check (SuiteQL `LIKE` on `companyname`) warns
before creating a customer that looks like it might already exist, but never
blocks submission.

## Deploy

No build step — it's a single `index.html`. Enable GitHub Pages on this repo
(root, `main` branch) and it's live at `https://lambwright.github.io/scout-intake/`.
