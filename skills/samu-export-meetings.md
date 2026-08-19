---
name: Export a date range of Samu meetings
description: >-
  Page through every meeting in a date range with its Samu Score, extractor output
  and CRM deal, for a warehouse sync, a coaching report or a forecast model.
api: openapi/samu-openapi.yml
operations:
  - GET /api/meetings
  - GET /api/users
  - GET /api/meeting/{id}/transcription
generated: '2026-08-13'
method: generated
source: openapi/samu-openapi.yml
---

# Export a date range of Samu meetings

This is the "API Out" flow Samu sells on the Enterprise plan: pull notes, insights
and metrics out of Samu into another system.

## Before you start

- Auth: account key in the **`apiKey`** header. Base URL `https://api.samu.ai`.

## Steps

1. **Resolve the roster once.**
   `GET /api/users` gives you `id → name/email` for the account. Meetings return
   participants as provider id strings, so cache this map before you start or the
   export is unreadable.

2. **Page the range.**
   `GET /api/meetings` requires **both** `dateFrom` and `dateTo`.
   - The range is capped at **366 days** from `dateFrom`. Split anything longer.
   - Date filters are **day-granular and inclusive** — they snap to the start and
     end of the given day in UTC, so a time component is effectively ignored.
   - Paginate with `limit` (max **500**, default 500) and `offset` (max **10000**).
   - Read the total from the **`X-Total-Count`** response header. The body is a
     bare JSON array, not an envelope, so there is nowhere else to get it.

3. **Stop before the offset ceiling.**
   With `offset` capped at 10000 and `limit` at 500, the deepest record you can
   reach in a single range is roughly the 10,500th. If `X-Total-Count` exceeds
   that, **narrow the date window and re-page** rather than incrementing offset —
   there is no cursor on this operation to escape with.

4. **Pull transcripts only where you need them.**
   `GET /api/meeting/{id}/transcription` is one call per meeting and returns every
   line (`{text, date, speaker}`). Do not fan this out across a full export
   unthinkingly — see the rate-limit note below.

## Fields worth mapping

- `score` → `{evaluables, score, feedback}`: the Samu Score and its rubric result.
- `extractor` → free-form object of the custom properties **your** account was
  configured to extract (competitor mentions, company size, and so on). Keys are
  account-specific and are not described in the spec; discover them from live data.
- `deal` → `{id, name, amount, stage}`, the linked CRM opportunity. This is the
  join key to HubSpot/Pipedrive/Salesforce; there is no deal endpoint in this API.
- `provider` → the origin channel, one of `GOOGLE`, `HUBSPOT`, `MICROSOFT`,
  `ZOOM`, `AIRCALL`, `ANURA`, `LAYER7`, `OFFLINE`, `IVR`, `MOBILE`.

## Rules and gotchas

- **`GET /api/meetings` is the one operation that declares a 429** ("Rate limit
  excedido"). No quota, window, `Retry-After` or `RateLimit-*` header is
  documented, so back off exponentially with jitter on any 429 and do not assume
  a reset time.
- `users[]` and `stakeholders[]` come back as **arrays of provider id strings** on
  read, even though they are objects on write. Resolve them through the roster
  from step 1.
- Errors are `{"status":"error","message":"..."}`. Only 400 is declared on this
  operation.
