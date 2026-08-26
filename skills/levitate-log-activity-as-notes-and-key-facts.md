---
name: levitate-log-activity-as-notes-and-key-facts
description: Push activity from an outside tool onto the Levitate record your team actually reads — notes attached to contacts and companies, and key facts that drive Levitate's timely outreach.
api: Levitate Public API
base_url: https://api.levitate.ai/public/v1
operations:
  - CreateNote
  - ListNotes
  - GetNoteById
  - DeleteNote
  - ListContactKeyFacts
  - AddContactKeyFact
  - GetContactKeyFactById
  - ReplaceContactKeyFact
  - DeleteContactKeyFact
generated: '2026-08-25'
method: generated
source: openapi/levitate-public-v1-openapi.json + https://help.levitate.ai/article/735-levitate-public-api
---

# Log activity into Levitate as notes and key facts

Two different write surfaces, two different purposes. **Notes** record what happened. **Key facts**
record what is durably true about a person — a birthday, an anniversary, a hobby — which is what
Levitate's outreach engine schedules against.

## Notes — `CreateNote` (`POST /public/v1/Notes`)

Required:
- `body` — an **HTML fragment**. Do not include the `<lev-content>` wrapper; it is added for you and is
  the only permitted root.
- at least one `reference` — `{ "type": "contact" | "company", "id": "..." }`. A reference that is not a
  live, visible contact or company on the account is rejected. **Up to 25 references per note.**

Optional: `visibility` — `shared` or `private`, defaulting to the caller's default.

The body is sanitized against a fixed allowlist. Anything outside it is **stripped silently** — the
note is not rejected — so send clean markup:
- Inline: `b i u s strike em strong small big sub sup abbr acronym cite code dfn kbd samp var q tt bdo font span`
- Blocks: `p div section article address center blockquote pre h1`–`h6` `hr br`
- Lists: `ul ol li dl dt dd` · Tables: `table caption col colgroup thead tbody tfoot tr th td`
- Links/media: `a img map area ins del`

`a` keeps only `http`, `https`, `mailto` and `tel` hrefs (`javascript:` is stripped). `img` sources
must be an `https` URL or an inline base64 data URI. Event attributes like `onclick` are always dropped.

Side effects to expect: logging a note may fan out to the contact timeline, Keep-in-Touch, and any
connected CRM. It is not a silent write.

Reading back: `ListNotes` filters on `contactId`, `companyId`, `createdAfter`/`createdBefore`,
`updatedAfter`/`updatedBefore`, with the standard `limit`/`pageToken`/`sort`. Rows are `NoteSummary`
with a `preview`; follow `url` or call `GetNoteById` for the full body.

Deleting: `DeleteNote` soft-deletes — the note leaves all reads immediately. A second delete returns
`404`. **No restore operation is published**, so treat it as final.

## Key facts — `/public/v1/Contacts/{id}/keyfacts`

- `AddContactKeyFact` requires a `type` and at least a `value` or a `date`. Given only a `date`, a
  display `value` is synthesized from it.
- A contact may hold at most **100** key facts.
- `ReplaceContactKeyFact` (`PUT .../keyfacts/{keyFactId}`) replaces one outright.
- `DeleteContactKeyFact` removes one — **except** a fact whose `readOnly` flag is set, which came from
  an official Levitate integration and cannot be modified or deleted through the API at all.
- `ListContactKeyFacts` / `GetContactKeyFactById` read them back. Note that the `keyFacts` array on the
  contact profile itself is read-only; all writes go through these endpoints.

## Errors

`400` — note with no body or no reference, key fact with neither value nor date, unknown or read-only
field. `403` — scope present but the acting user lacks the permission. `404` — the parent contact is not
visible to the caller, the record does not exist, or a note was already deleted; Levitate deliberately
does not distinguish "not visible" from "does not exist", so never infer existence from a `404`.
`401` — missing, expired or wrong-surface credential.

Every response is an `OperationResult`; log `requestId` when escalating.
