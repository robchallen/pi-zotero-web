# pi-zotero-web: group libraries are unreachable through the `zotero_*` tools

Repo: https://github.com/robchallen/pi-zotero-web (checked at `f81acb1`)

## Summary

Group libraries are readable from both the Zotero web API and the local desktop API,
but there is no way to reach them through the `zotero_*` tools. Group support already
exists in the client layer; it is just not reachable from a tool call.

The agent can only answer "what is in my group library?" by bypassing its own tools and
calling the API directly with `curl`, which also means reading the stored API key out of
`~/.pi/agent/auth.json`.

## What works today

| Route | Group access |
|---|---|
| Web API, `GET https://api.zotero.org/groups/<id>/items/top` (API key header) | works |
| Local API, `GET http://localhost:23119/api/groups/<id>/items/top` (no key) | works |
| `zotero_collection` / `zotero_search` / `zotero_item` etc. | personal library only |

## Why

Group support is implemented but is a global, credential-level switch:

- `client.ts:42` — `if (cfg.groupId) return "/groups/" + cfg.groupId;` so every request path
  already handles groups correctly.
- `tools.ts:61-62`, `95-96`, `119-120` — `cfg.groupId` is populated *only* from
  `auth.env.ZOTERO_GROUP_ID`.
- No tool input schema exposes a `groupId` / `library` parameter, so an agent cannot select
  a library per call.
- The provider writes `ZOTERO_USER_ID` (web, `provider.ts:203-205`) or `ZOTERO_MODE=local` +
  `ZOTERO_BASE_URL` (local, `provider.ts:162-164`) but never `ZOTERO_GROUP_ID`, so there is
  no supported path to set it other than hand-editing `auth.json`.
- There is no tool to *discover* group ids, so even with a config switch the user would have
  to supply the numeric id by hand.

Net effect: one library per credential, fixed at auth time, with no discovery.

## Suggested fix

1. Add an optional `library` (or `groupId`) parameter to the read tools
   (`zotero_search`, `zotero_collection`, `zotero_item`, `zotero_attachment`, `zotero_tags`,
   `zotero_fulltext`, `zotero_export`), merged into `cfg` alongside the existing
   `ZOTERO_GROUP_ID` fallback. `client.ts` needs no change.
2. Add a `zotero_groups` tool (or a `groups` action on an existing tool) so ids are
   discoverable rather than hand-entered.
3. Decide the write story separately — defaulting writes to a group the user did not name
   seems worth avoiding, so group writes could stay opt-in.

### Local API calls a `zotero_groups` tool would need

Verified against a running Zotero 10.0.1 desktop (`X-Zotero-Version: 10.0.1`,
`Zotero-API-Version: 3`, `Zotero-Schema-Version: 44`):

| Call | Status | Note |
|---|---|---|
| `GET /api/users/0/groups` | 200 | returns the group list |
| `GET /api/groups` | 404 | not a valid local endpoint |
| `GET /api/groups/<id>/collections` | 200 | |
| `GET /api/groups/<id>/items/top?limit=100` | 200 | |

So the discovery call is `GET /api/users/0/groups` locally, and
`GET /users/<userID>/groups` on the web API — same path shape, so it fits the existing
`prefix()` + `getBaseUrl()` plumbing.

One local-vs-web difference worth handling: locally, `data.type` comes back as `null`,
whereas the web API returns `"Private"` / `"PublicOpen"`. A `zotero_groups` tool should
not depend on `type` being populated.

## Reproduction

Stored credential (redacted): `{ type: "api_key", key: "<redacted>", env: { ZOTERO_USER_ID: "3444309" } }`
— note no `ZOTERO_MODE`, so `isLocal()` is false and the tools use the web API.

Library used for testing: group `6686137` ("group-api-test", private), which contains one
top-level item plus two attachments and no collections.

1. `zotero_collection { action: "list", top: true }` → lists personal-library collections only;
   the group is absent and there is no parameter that would include it.
2. `curl -H "Zotero-API-Key: $KEY" https://api.zotero.org/groups/6686137/items/top` → returns the item.
3. `curl http://localhost:23119/api/groups/6686137/items/top` → returns the same item, plus
   attachment links; `…/tags` and `…/collections` also work, unauthenticated.

## Information available on request

- `zotero_*` tool schemas as exposed to the agent.
- Full JSON responses from any of the local or web API calls above.
- Redacted `auth.json` credential shape; local API response headers.
- Tests against other local endpoints on Zotero 10.0.1 if useful for scoping.
