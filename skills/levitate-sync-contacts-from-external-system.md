---
name: levitate-sync-contacts-from-external-system
description: Create or update Levitate contacts from records held in another system (CRM, AMS, spreadsheet, web form), setting tags and custom fields in the same call, without destroying data already on the record.
api: Levitate Public API
base_url: https://api.levitate.ai/public/v1
operations:
  - ListContacts
  - SearchContacts
  - GetContactById
  - CreateContact
  - UpdateContact
generated: '2026-08-25'
method: generated
source: openapi/levitate-public-v1-openapi.json + https://help.levitate.ai/article/735-levitate-public-api
---

# Sync contacts into Levitate from another system

Levitate's Public API is **not a connector**. It never reaches out to a CRM or AMS to pull records —
your code reads from the source system and writes into Levitate. This skill is that write path.

## Before you start

- The Public API must be enabled on the account by a Levitate Success Specialist. If it is not, no key can be generated.
- Send `Authorization: Bearer <token>` on every request — a Personal API Key or an OAuth 2.0 (authorization code + PKCE) token.
- The credential needs the `levitate:contacts` scope. That scope covers contacts, companies, notes **and** key facts.
- A key scoped to the Levitate MCP Server, or a Zapier account key, will **not** work here. It returns `401 You must be authenticated to perform this operation.`

## Steps

1. **Find out whether the contact already exists.** Call `SearchContacts`
   (`POST /public/v1/Contacts/search`) with a filter on `email` / `equals`, or `ListContacts`
   (`GET /public/v1/Contacts?email=...`) for a simple lookup. Both return lean `ContactSummary` rows
   with an `id` and a `url`.

2. **Create when absent.** Call `CreateContact` (`POST /public/v1/Contacts`). The payload needs at
   least one email address, or both a first and last name. Set `tags` and `customFields` in the same
   request so the contact lands already categorized. Tag names that do not exist yet are created
   automatically. Expect `201`.
   - Do **not** try to set `company` — it is inferred from the primary email's domain and is rejected as read-only.
   - Other read-only fields that will be rejected: `source`, `owner`, `emailSubscribed`, `textSubscribed`, `lastCommunicationDate`, `keyFacts`.

3. **Handle `409 Conflict`.** A create whose primary email already exists on an account contact returns
   `409` and creates nothing. Treat this as "already present": look the contact up and fall through to
   step 4. This is the only duplicate protection the API offers — there is no `Idempotency-Key` header,
   so a retry after a network timeout also returns `409`, not the original `201` body. Never assume a
   `409` means your first attempt failed.

4. **Update when present — read the arrays first.** Call `UpdateContact`
   (`PATCH /public/v1/Contacts/{id}`). The semantics that bite:
   - An omitted field is left unchanged.
   - An explicit `null` clears a scalar.
   - **A supplied array replaces the entire array.** Sending `tags` overwrites every tag on the contact.
     The same is true of `emailAddresses`, `phoneNumbers` and `customFields`.
   So to add one tag: `GetContactById`, take the current `tags`, append yours, and PATCH the full list
   back. There is no restore and no prior version — an overwrite here is unrecoverable.
   - The first entry of `emailAddresses` / `phoneNumbers` is the primary.

5. **Page through anything large.** List and search return `limit` 25 by default, 100 maximum. Follow
   `pageToken` from the `PagedCollectionOf...` envelope until it is null.

## Error handling

Every response is wrapped in `OperationResult` — check `success`, read `systemMessage`, and log
`requestId` for support.

| Status | Meaning here | What to do |
|---|---|---|
| `400` | Unknown or read-only field, or a contact with neither an email nor a full name | Fix the payload; nothing was written |
| `401` | Missing/expired key, or a key for the wrong Levitate surface | If `systemMessage` names an expiry date, the key is dead — expiry is final, generate a new one |
| `403` | Scope missing, or the acting user's own Levitate permission does not allow it | Both checks must pass; widening the scope alone will not help |
| `409` | Duplicate primary email | Look up and PATCH instead |

## Irreversibility warning

`public-v1` publishes **no** `DELETE` for a contact. A contact you create through the API cannot be
removed through the API. Nothing on this surface has an undo, restore or recovery window. Confirm
before you write.
