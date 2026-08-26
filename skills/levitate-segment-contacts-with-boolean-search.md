---
name: levitate-segment-contacts-with-boolean-search
description: Build a precise Levitate audience with the nested AND/OR/NOT contact search, then page the whole result set — for outreach targeting, reporting, or lapsed-relationship discovery.
api: Levitate Public API
base_url: https://api.levitate.ai/public/v1
operations:
  - SearchContacts
  - ListContacts
  - GetContactById
  - ListCompanies
generated: '2026-08-25'
method: generated
source: openapi/levitate-public-v1-openapi.json (SearchContacts description) + https://help.levitate.ai/article/735-levitate-public-api
---

# Segment Levitate contacts with the boolean search endpoint

`ListContacts` combines its query-string filters with AND only. Anything needing OR, NOT, or grouping
goes to `SearchContacts` (`POST /public/v1/Contacts/search`). Both return the same lean
`ContactSummary` rows and use the same paging and sort parameters.

## The filter shape

The body is a single `filter` node. A node is either:

- a **group** — `{ "op": "and" | "or" | "not", "filters": [ ... ] }`
  (`and` / `or` take one or more children, `not` takes exactly one), or
- a **condition** — `{ "field": ..., "operator": ..., "value": ... }`

Every `value` is sent as a **string**, including booleans and dates.

## Fields and their operators

An unknown field, or an operator not valid for that field, returns `400`.

| Field | Operators |
|---|---|
| `name` | `contains` (case-insensitive prefix match on display name) |
| `email` | `equals` (matches any of the contact's addresses) |
| `company` | `equals` (by company id) |
| `companyName` | `contains` |
| `tags` | `contains`, `notContains` |
| `city`, `stateProvince` | `equals`, `startsWith` |
| `postalCode` | `startsWith` |
| `source` | `equals` |
| `createdAt`, `updatedAt` | `before`, `after`, `on` (date-only `YYYY-MM-DD`; `on` matches the whole day) |
| `lastCommunicationDate` | `before`, `after`, `on`, `isNull` |
| `customFields.{name}` | `equals`, `isEmpty`, `isNotEmpty` (the field must be defined on the account) |
| `visibility` | `equals` (`shared` \| `private`) |
| `emailSubscribed`, `textSubscribed`, `hasEmail`, `hasPhone` | `equals` (boolean as a string) |

## Hard limits

- The tree may nest at most **5 levels** deep.
- The tree may hold at most **50 conditions**.
- Page size: `limit` default 25, maximum 100.

## Steps

1. **Resolve any company filter first.** `company` matches on company **id**, not name. Call
   `ListCompanies` (`GET /public/v1/Companies?name=...`) to get the id, or filter on `companyName`
   with `contains` if a prefix match is good enough.
2. **Compose the tree.** Example — everyone in NC tagged `Client` but not `Churned`, with no contact
   since a date:
   ```json
   { "filter": { "op": "and", "filters": [
       { "field": "stateProvince", "operator": "equals", "value": "NC" },
       { "field": "tags", "operator": "contains", "value": "Client" },
       { "field": "tags", "operator": "notContains", "value": "Churned" },
       { "field": "lastCommunicationDate", "operator": "before", "value": "2026-02-25" }
   ] } }
   ```
3. **Sort deliberately.** `sort` accepts `creationDate` (default, descending) or `name`; prefix with
   `-` to reverse. Keep the sort stable across pages.
4. **Page to exhaustion.** Send `limit=100`, then follow `pageToken` from the response envelope until
   it is null. `totalCount` tells you how many rows to expect.
5. **Hydrate only what you need.** Rows are summaries. Follow the row's `url`, or call
   `GetContactById`, when you need the full profile with key facts and custom fields.

## Errors

`400` on an unknown field, a bad operator/field pairing, an over-deep or over-wide tree, or a
malformed date. `403` when the acting user's own Levitate permissions do not reach those contacts —
the result set is always bounded by what that user can see in the app. `401` on a missing, expired or
wrong-surface credential.

## Note on the MCP alternative

The `contacts_search` tool on Levitate's MCP server covers the same ground conversationally, and
`companies_search` adds AND/OR/EXCLUDE tag logic for companies that this REST endpoint does not
expose. See `mcp/levitate-tool-crosswalk.yml`.
