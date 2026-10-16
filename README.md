# Veekay Cabs CRM — Phase 1 (Foundation)

A separate, self-contained application from the main `client`/`server`
(its own frontend, backend, auth and logins), but connected to the **same
MongoDB database** as the main site — so it can look up a customer's
existing account and bookings (for duplicate detection / customer history)
without needing any sync layer.

## Structure

- `server/` — Express + MongoDB API (port 5050)
- `client/` — Next.js (TypeScript) frontend (port 4100)

## First-time setup

```bash
cd crm/server && npm install
cd ../client && npm install
```

`crm/server/.env` already has `MONGODB_URI` copied from the main server's
`.env` (same database) and a fresh, independent `JWT_SECRET` — CRM logins
are not interchangeable with the main site's admin logins.

Create the first Super Admin login:

```bash
cd crm/server
npm run seed:admin -- --email you@veekaycabs.com --password YourPassword --name "Your Name"
```

## Running it

```bash
cd crm/server && npm run dev   # http://localhost:5050
cd crm/client && npm run dev   # http://localhost:4100
```

## What's built

- Super Admin / Salesperson login (JWT, separate from main site)
- Salesperson management (create, edit, deactivate/reactivate)
- Lead CRUD with the full SRS status pipeline
- **Round Robin auto-assignment** — every new lead, from any source, is
  assigned automatically to whichever active salesperson has gone longest
  without a new one (falls back to Super Admin if no salesperson is active)
- **15-minute SLA + auto-reassignment** — a background job (`node-cron`,
  runs every minute) reassigns any lead whose owner hasn't logged a call or
  WhatsApp within 15 minutes of assignment to the next salesperson in
  rotation, and keeps a lifetime `slaBreachCount` per lead for reporting
- **"Not Interested" auto-transfer** — a salesperson marking a lead "Not
  Interested" hands it to Super Admin for review automatically; the full
  history (who marked it, the reason, who had it before) stays on the
  timeline
- **WhatsApp alert to the owner** on every automatically-captured lead
  (Website, Meta Ads, JustDial) — reuses the same AiSensy account the main
  site already sends from, as a free-text session message (works as long as
  you occasionally message the business number yourself — see note below)
- Activity timeline (calls, WhatsApp, notes, status changes, assignment
  history) — a call/WhatsApp log is what "stops" the SLA per the SRS
- Duplicate-lead flagging by mobile number (flagged, never silently dropped)
- Customer history lookup — cross-references the main site's `users`/
  `bookings` collections by mobile number (read-only, never writes there)
- Dashboard KPI cards, role-scoped (a salesperson only ever sees their own
  leads and numbers — enforced server-side, not just hidden in the UI)

## Lead sources — what's live vs what needs your setup

| Source | Status |
|---|---|
| **Manual** (typed in by Super Admin/salesperson) | ✅ Live — "Add Lead" button |
| **Website** (Contact Us form) | ✅ Live — `client/app/contact/ContactClient.tsx` now actually submits to `POST /api/public/leads` instead of the old fake 1.5s delay that never saved anywhere |
| **Google Ads** (click-through to website) | ✅ Live — if the visitor's URL has `utm_source=google`, the lead is auto-tagged "Google Ads" with the full UTM set stored |
| **Google Ads** (native Lead Form extensions) | 🟡 Code is ready (`/api/webhooks/google-ads`), needs **your** setup (below) — direct from Google, no API/OAuth needed, does not touch NeoDove |
| **Meta Ads** (Facebook/Instagram Lead Ads) | 🟡 Code is ready (`/api/webhooks/meta` — handshake + leadgen payload → Graph API fetch → Lead, auto-assigned + WhatsApp alert), needs **your** Meta-side setup (below) — direct from Meta |
| **JustDial** | 🟡 Code is ready (`/api/webhooks/justdial`), needs a way to actually get JustDial's lead notifications there — see below, there's no localhost-testable way around this one — direct from JustDial (via a Zapier relay) |
| **Phone Call / IVR** (via NeoDove — this is the *only* NeoDove touchpoint) | 🟡 Code is ready (`/api/webhooks/neodove`), needs **your** NeoDove-side webhook setup (below) — NeoDove is used purely as the IVR call engine, not as a CRM; we don't use or depend on any other NeoDove feature |
| **WhatsApp** (inbound enquiries) | ⬜ Not built — skipped for now per your call |

**Architecture note:** every source integrates *directly* with our own CRM —
Google Ads, Meta Ads, JustDial, and the website all post straight here, none
of them pass through NeoDove. NeoDove is kept only because that's where the
IVR phone system itself lives (incoming customer calls have to be answered
somewhere) — its webhook is used solely to pull call/IVR details into our
CRM, nothing else. We never read from or write to NeoDove's own lead
pipeline, pipelines, or dashboard.

### Meta Ads setup (to actually go live)

1. Deploy `crm/server` somewhere with a public HTTPS URL (webhooks can't reach `localhost`).
2. In [Meta for Developers](https://developers.facebook.com), create an App → add the **Webhooks** product → Page → subscribe to the **leadgen** field, pointing at `https://<your-domain>/api/webhooks/meta`.
3. Pick any random string as your verify token, put it in `META_WEBHOOK_VERIFY_TOKEN` in `crm/server/.env`, and enter the same string in Meta's dashboard when it asks to verify the URL.
4. Generate a Page Access Token for the Facebook Page running the ads (Graph API Explorer or a System User token), put it in `META_PAGE_ACCESS_TOKEN`.
5. Connect your Lead Ads forms to that Page if not already. New submissions will then create Leads automatically with source "Meta Ads".

### NeoDove setup (to actually go live) — IVR/call leads only

NeoDove is used **only** as the IVR calling engine here — not as a lead
management CRM, and not as a relay for Google Ads/Meta/JustDial leads. The
only thing needed from NeoDove is for it to forward call details to our CRM
when a customer calls through the IVR:

1. Deploy `crm/server` somewhere with a public HTTPS URL.
2. In NeoDove's dashboard, find the webhook/integration settings. On the
   screenshot you sent (Settings → **Manage Columns**), that's a different
   tab — it only controls which columns show in NeoDove's own lead list, not
   where data gets sent. Click the **`>`** arrow next to the tabs (Users,
   Preference, Accessibility, Pipelines, Profile, Roles and Permission,
   Automatic Report, Manage Columns) to scroll right — the integration/API/
   webhook tab (if NeoDove exposes one) will be further along that row, or
   possibly under **Cloud Telephony** in the left sidebar since that's their
   call-infrastructure section. If you can't find it, ask NeoDove support
   directly for "outbound webhook / API on new call", since this isn't
   always self-serve on every plan.
3. Point the outbound webhook at `https://<your-domain>/api/webhooks/neodove`.
4. Optional: set `NEODOVE_WEBHOOK_TOKEN` in `crm/server/.env` and the matching value on NeoDove's side, if they support a custom header/query param — this stops anyone else from posting fake leads to that URL.
5. The receiver accepts several common field-name variants already (phone, name, call status, duration, recording URL, agent). If NeoDove's actual payload uses different names, the server log prints the full raw payload on every call so the mapping can be adjusted precisely. It also dedupes safely: if NeoDove includes its own call ID in the payload, a retried delivery (e.g. after our server was briefly down) is detected and skipped instead of creating a second lead.

### Google Ads Lead Form setup (to actually go live) — direct, not via NeoDove

1. Deploy `crm/server` somewhere with a public HTTPS URL.
2. In Google Ads, go to **Assets → Lead form** → open (or create) your lead form asset.
3. Under **Lead delivery**, choose **Webhook integration**.
4. Enter Webhook URL: `https://<your-domain>/api/webhooks/google-ads`, and set a Webhook key (any random string) — put that same string in `GOOGLE_ADS_WEBHOOK_KEY` in `crm/server/.env`.
5. Google sends a test payload when you save — it's auto-acknowledged (logged as "Test payload acknowledged"), so a successful save confirms the connection is working.

### JustDial setup (to actually go live)

JustDial has no public webhook for most business listings (only certain
paid/API-enabled accounts do) — the realistic no-code path is a relay:

1. Deploy `crm/server` with a public HTTPS URL.
2. Check your JustDial dashboard for **API access / Lead Sync / CRM
   Integration** — if you have it, point it straight at
   `https://<your-domain>/api/webhooks/justdial` and skip the rest.
3. Otherwise: JustDial emails you on every new lead. Set up a free
   [Zapier](https://zapier.com) flow — trigger "New matching Email" (Gmail,
   filtered to JustDial's sender address) → action "Webhooks by Zapier →
   POST" → `https://<your-domain>/api/webhooks/justdial`, mapping the
   customer name/phone/query out of the email into the JSON body. Zapier's
   free tier covers low volume.
4. Optional hardening: set `JUSTDIAL_WEBHOOK_TOKEN` in `crm/server/.env` to
   a random string and add the same value as `?token=...` in the Zapier
   webhook URL, so nobody else can post fake leads to that endpoint.

### A note on the WhatsApp owner-alert

It sends from the same AiSensy account/number the main site already uses.
AiSensy only allows **free-text** messages to a number that has an open
24-hour WhatsApp conversation with your business number — so it works as
long as you (or whoever reads `ADMIN_MOBILE`'s messages) occasionally send
the business WhatsApp number a message yourselves. If it ever needs to be
fully guaranteed-delivery regardless of that, it would need a pre-approved
WhatsApp template (a Meta approval step, same as the booking ones).

## Not built yet

- **Real Booking conversion** — "Booking Confirmed" is currently just a
  status + a free-text `bookingReference` field; actually creating the
  booking in the main site's database from here isn't wired up yet
- **Proper reports/exports** beyond the dashboard KPI tiles
- **Google Lead Form extensions, WhatsApp inbound, Phone/IVR** — see table above
