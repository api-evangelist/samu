---
name: Read Samu WhatsApp and chat conversation threads
description: >-
  Walk the chat inbox — WhatsApp, HubSpot and email threads — down to individual
  messages and Samu's per-day AI summaries and action items.
api: openapi/samu-openapi.yml
operations:
  - GET /api/chat/threads
  - GET /api/chat/threads/{threadId}
  - GET /api/chat/threads/{threadId}/messages
  - GET /api/chat/threads/{threadId}/interactions
generated: '2026-08-13'
method: generated
source: openapi/samu-openapi.yml
---

# Read Samu WhatsApp and chat conversation threads

WhatsApp is the dominant B2B sales channel in Samu's market, and this is the
surface that exposes it. Three levels: threads, the messages inside them, and the
daily AI interaction snapshot Samu derives from them.

## Before you start

- Auth: account key in the **`apiKey`** header. Base URL `https://api.samu.ai`.

## Steps

1. **List the inbox.**
   `GET /api/chat/threads` returns `{items[], nextCursor}`.
   - Filter with `provider` (`WHATSAPP`, `HUBSPOT`, `EMAIL`), `threadType`
     (`dm`, `group`), and `dateFrom`/`dateTo` against the thread's
     `lastMessageAt`.
   - Paginate with **`cursor`** (a date-time) and `limit`. Follow `nextCursor`
     until it comes back `null`.
   - Each item carries `owner` (the Samu user who owns the thread), `title` (the
     group name for `group`, the contact name for `dm`), `lastMessageAt` and a
     `contacts[]` array.
   - **Keep `contacts[]`.** You need it in step 3.

2. **Fetch one thread** with `GET /api/chat/threads/{threadId}` when you already
   hold an id. Same shape as a list item.

3. **Page the messages.**
   `GET /api/chat/threads/{threadId}/messages` returns `{items[], nextCursor}`.
   - Pagination here uses **`before`**, not `cursor` — a different parameter name
     from step 1 for the same idea. `from`/`to` additionally bound the history by
     day, which is how you avoid dragging years of a long WhatsApp thread.
   - Each message has `direction` (`inbound`/`outbound`), `sentAt`, `content`
     (`{type, text}` where type is `text`/`audio`/`image`/`video`/`document`),
     `attachments[]` and a `sender` of `{kind, contactId, userId}` where `kind` is
     `contact`, `user` or `provider_identity`.
   - **Sender names are not denormalized.** The spec says to cross-reference
     `sender.contactId` against `thread.contacts` from step 1 to get a name.

4. **Read the daily AI snapshot.**
   `GET /api/chat/threads/{threadId}/interactions` returns
   `{items[], total, page, perPage}` — one item per day of activity, carrying
   `summary` (Samu's long or short summary of that day), `extractor` (your custom
   extracted properties for that day), `actionItems[]`
   (`{id, description, status, dueAt, resolvedAt}`) and a processing `status`.
   - This one paginates with **`page`/`perPage`** (default 50, max 200) and gives
     you a `total` — a third pagination style again.
   - `from`/`to` filter by day, inclusive.
   - Fields prefixed `samu_*` are internal and are excluded from `extractor`.

## Rules and gotchas

- **Three pagination styles across these four operations**: `cursor` on threads,
  `before` on messages, `page`/`perPage` on interactions. Write one adapter per
  surface; there is no shared envelope.
- All date filters are **day-granular and inclusive in UTC** — passing a precise
  timestamp does not narrow the window within a day.
- Interactions are a **daily rollup**, not a per-message analysis. To correlate a
  summary with what was actually said, pull the same day's messages with
  `from`/`to` set to that date.
- None of these four operations declares any error response at all — not even the
  400 the meetings endpoints declare. Handle non-2xx defensively; the envelope is
  `{"status":"error","message":"..."}`.
- Nothing in the published contract links a chat thread to a meeting or to a CRM
  deal. The two domains cannot be joined through this API.
