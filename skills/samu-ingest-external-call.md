---
name: Ingest an externally recorded call into Samu
description: >-
  Push a call that Samu did not record itself (a dialer, IVR, mobile or offline
  recording) into Samu for transcription and AI analysis, then read back the
  transcript once processing completes.
api: openapi/samu-openapi.yml
operations:
  - POST /api/meeting
  - GET /api/meeting/{id}
  - GET /api/meeting/{id}/transcription
  - GET /api/users
generated: '2026-08-13'
method: generated
source: openapi/samu-openapi.yml
---

# Ingest an externally recorded call into Samu

Samu records WhatsApp, phone, video and Teams/Meet conversations on its own. This
flow is for the other case: audio or video your system already holds, which you
want Samu to transcribe, score and attach to a CRM deal.

## Before you start

- Auth: send the account key in a header named **`apiKey`**. Not `Authorization`,
  not `X-API-Key`. Every operation below requires it.
- API access is an **Enterprise-plan** feature; there is no self-serve key.
- Base URL: `https://api.samu.ai`

## Steps

1. **Confirm the participants exist in Samu.**
   `GET /api/users` returns the account's users (`id`, `name`, `email`, `enabled`,
   `lang`). `hostEmail` on the meeting you create **must** be the email of a
   registered Samu user, and so must every entry in `users[]`. Check here first —
   the API joins participants by email, not by id, so a typo silently produces a
   meeting with no owner.

2. **Create the meeting.**
   `POST /api/meeting` with a JSON body. Required: `provider`, `dateFrom`,
   `media`. Also send `hostEmail` so the call is attributed.
   - `provider` must be one of `GOOGLE`, `HUBSPOT`, `MICROSOFT`, `ZOOM`,
     `AIRCALL`, `ANURA`, `LAYER7`, `OFFLINE`, `IVR`, `MOBILE`. Use `OFFLINE` for
     an in-person recording, `IVR`/`MOBILE` for telephony.
   - `media` is a **publicly reachable URL** to an `.mp4` or `.mp3`. Samu
     downloads the file and re-hosts it; this is not a multipart upload. A signed
     URL must stay valid long enough for that fetch.
   - Omit `eventId`/`conferenceId` and Samu generates them. Omit `dateTo` and it
     defaults to `dateFrom`.
   - `users[]` and `stakeholders[]` take objects (`email`, `name`, `lastName`,
     `phone`, `providerId`); both may be empty arrays.
   - Send `transcription` only if you already have one; otherwise Samu transcribes
     the media itself.
   - Optional `location` takes `latitude`/`longitude` for field sales.
   The 200 response is `{"status":"ok"}` shaped and carries the new meeting id.

3. **Wait, then read the meeting back.**
   `GET /api/meeting/{id}` returns the `Meeting`: `duration`, `score`
   (`{evaluables, score, feedback}` — the Samu Score), `extractor` (your
   account's custom extracted fields), `callType` and `deal`
   (`{id, name, amount, stage}` from the CRM).

4. **Read the transcript.**
   `GET /api/meeting/{id}/transcription` returns an array of lines, each
   `{text, date, speaker}`. Note this is **not** the same shape you may have sent
   on write — the write-side `Transcription` object uses `messages[]` with
   `participantId`/`startAt`/`endAt` plus a `participants` map. The read side is
   flattened with speaker names resolved.

5. **Correct metadata later if needed.**
   `PUT /api/meeting/{id}` updates `name`, `hostEmail`, `stakeholders`, `dateFrom`
   and `dateTo`. It cannot change `media` or `provider`.

## Rules and gotchas

- **There is no idempotency key.** `POST /api/meeting` is not safe to blind-retry:
  a retry after a timeout creates a second meeting and a second media download and
  transcription. Generate your own stable `eventId` and, before retrying, list
  meetings for the day (`GET /api/meetings` with `dateFrom`/`dateTo`) and check
  whether yours already landed.
- **Processing is asynchronous and there is no callback.** The docs say the video
  "tardará unos minutos en ser subido". No webhook, no status field on the meeting
  and no polling operation is published. Poll `GET /api/meeting/{id}` on a backoff
  and treat a missing `score`/`extractor` as "not ready yet".
- **Errors are a flat envelope**, `{"status":"error","message":"..."}` — not RFC
  9457. Only 400 and 404 are declared; a bad or missing `apiKey` is undeclared, so
  handle a 401 defensively.
- `GET /api/meeting/{id}` is the only operation that declares a 404. Everything
  else surfaces problems as a 400.
