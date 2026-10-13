# pi-zotero-web — as-is scope

Describes the code as it stands at `f81acb1` (v0.2.2). Written by reading the source, not
the README; where the two disagree it is noted. Nothing here is a proposal — proposals are
confined to the last section, which is explicitly marked as such.

Size: 1877 lines of source (`client.ts` 861, `tools.ts` 736, `provider.ts` 252, `index.ts` 28)
and 1435 lines of test (90 tests, all passing).

## 1. What it is

A pi extension that gives the agent read/write access to a Zotero library, over either the
Zotero Web API (`https://api.zotero.org`) or the Zotero desktop app's Local API
(`http://localhost:23119/api`). It registers one auth provider and nine tools. It declares no
LLM models, so it never appears in `/model`.

Runtime dependencies: none of its own. It uses `fetch`, `node:crypto`, `node:fs/promises`,
`node:path`, and three host packages provided by pi (`@earendil-works/pi-coding-agent`,
`@earendil-works/pi-ai`, `typebox`). Dev dependency: `tsx`, for running the tests.

## 2. Files

| File | Role |
|---|---|
| `index.ts` | Entry point. Registers the provider, registers the tools, and notifies on `session_start` if a credential exists. |
| `provider.ts` | The `zotero` auth provider: the `/login zotero` flow, key verification, credential shape. Also `isLocalApiReachable()` and the two base-URL constants. |
| `client.ts` | All HTTP. One `zoteroFetch` wrapper plus ~30 exported functions, one per Zotero endpoint. Holds the local write-authorization state machine. |
| `tools.ts` | The nine agent-facing tools: TypeBox schemas, `resolveConfig()`, dedup-on-create, and output summarising. |

Dependency direction is one-way: `index.ts` → `tools.ts` → `client.ts` → `provider.ts`
(for the base-URL constants only).

## 3. Configuration and mode selection

### Credential

`/login zotero` (`provider.ts:120-200`) checks whether local Zotero answers on port 23119.

- **Local offered and chosen** — stores `{ type: "api_key", key: "local", env: { ZOTERO_MODE: "local",
  ZOTERO_SERVER_ID: <header>, ZOTERO_BASE_URL: "http://localhost:23119/api" } }`. No key is requested.
- **Web** — prompts for a key (up to 3 attempts), verifies it against `GET /keys/current`, requires
  `access.user.library`, and stores `{ key, env: { ZOTERO_MODE: "web", ZOTERO_USER_ID, ZOTERO_BASE_URL } }`.

If local Zotero is not running, the web branch is taken without a prompt.

### Resolution at call time

Every tool that touches a library calls `resolveConfig(ctx)` (`tools.ts:43-122`) afresh:

1. `ZOTERO_MODE === "web"` → web mode; throws if key or user id is missing.
2. Otherwise **probe `http://localhost:23119/api/`** (`isLocalApiReachable`). If it answers 200, or
   `ZOTERO_MODE === "local"`, or `key === "local"` → local mode, `userId` defaulting to `"0"`.
3. Otherwise → web mode; throws a `/login zotero` message if key or user id is missing.

Consequence: local Zotero wins whenever it is running, even for a user who configured a web key,
unless `ZOTERO_MODE` is explicitly `"web"`. Step 2 costs one localhost request per tool call.

### Environment knobs read from `auth.json`

`ZOTERO_MODE`, `ZOTERO_BASE_URL`, `ZOTERO_USER_ID`, `ZOTERO_SERVER_ID`, `ZOTERO_LOCAL_KEY`,
`ZOTERO_GROUP_ID`. Only the first four are ever written by `/login`; the last two must be
hand-edited. `ZOTERO_GROUP_ID` is the only way to reach a group library — see
`design/issue/local-group-libraries.md`.

## 4. Request layer (`client.ts`)

**Path prefix** (`client.ts:41-45`): `/groups/<groupId>` when `cfg.groupId` is set, else
`/users/<userId>` (`userId` defaults to `"0"` locally, `""` on web). Every library call is built
from this, so group support is global and credential-level, not per-call.

**Headers** (`client.ts:162-190`): always `Zotero-API-Version: 3`. On web, `Zotero-API-Key` on
every request. On local, reads send no key (they are unauthenticated by design); writes send
`Zotero-API-Key` plus `Zotero-Server-ID`.

**Local write authorization** (`client.ts:60-155`, `client.ts:190-215`) — a small state machine, the
most intricate part of the codebase:

- A module-level `localSessions: Map<baseUrl, { serverId, writeKey, remember }>` caches per base URL
  for the life of the process.
- Missing server id → `GET /` and read `Zotero-Server-ID`; this is also where the Zotero 10+ version
  check lives.
- Missing write key → `POST /local/authorize`, which raises the desktop Allow / Always Allow / Deny
  modal. 403 → "denied", 429 → "rate limited".
- "Allow" returns a single-use key (`remember: false`), discarded after one request. "Always Allow"
  returns a durable key and calls `cfg.saveAuth` — see section 7, this does not actually persist.
- 401 → clear the key and re-authorize once. 412 → clear server id *and* key and re-authorize once.

**Errors**: every non-2xx becomes `ZoteroError(message, status, body)`. Write preconditions use
`If-Unmodified-Since-Version`, so stale versions surface as 412 from Zotero.

## 5. Tools

Nine tools, all registered unconditionally in `registerZoteroTools` (`tools.ts:247-736`).
"Writes" means the action can modify the library.

| Tool | Actions | Writes | Client calls |
|---|---|---|---|
| `zotero_search` | — (`q`, `qmode`, `itemType`, `collectionKey`, `tag`, `limit`, `top`) | no | `searchItems` → `/items`, `/items/top`, or `/collections/<k>/items` |
| `zotero_item` | `get`, `create`, `update`, `delete` | yes | `getItem`, `createItems`, `updateItem`, `deleteItem` |
| `zotero_template` | — (`itemType`, `linkMode`) | no | `itemTemplate` → public `GET /items/new` |
| `zotero_attachment` | `list`, `upload`, `download`, `delete` | yes | `getChildren`, `uploadAttachmentFile`, `downloadAttachmentFile`, `deleteItem` |
| `zotero_tags` | `list`, `get`, `set` | yes (`set`) | `listTags`, `setItemTags` (a `PATCH` of the whole tags array) |
| `zotero_collection` | `list`, `get`, `items`, `create`, `rename`, `delete`, `add`, `remove` | yes | `listCollections`, `getCollection`, `listCollectionItems`, `createCollections`, `updateCollection`, `deleteCollection`, `add/removeItemsToCollection` |
| `zotero_export` | — (`format`, `itemKeys`, `collectionKey`) | no | `exportItems` → `/items?format=<fmt>` |
| `zotero_schema` | `itemTypes`, `itemFields`, `itemTypeFields`, `creatorTypes`, `creatorFields` | no | five public, unauthenticated endpoints |
| `zotero_fulltext` | `get`, `set` | yes (`set`) | `getFullText`, `setFullText` |

`zotero_template` and `zotero_schema` never resolve a config and never authenticate; they always
hit `https://api.zotero.org` even in local mode, so they need internet access regardless of mode.

Output is JSON text. Item-returning tools pass through `summarize()` (`tools.ts:124-139`), which
keeps only key, version, itemType, title, creators, abstractNote, date, DOI, url, tags, collections —
all other fields (journal, pages, publisher, extra, …) are dropped from tool output, though
`zotero_item action=get` returns the raw item untouched.

## 6. Behaviours worth knowing

**Dedup on create** (`tools.ts:166-235`, ~70 lines across four helpers). A single-item `create` first searches the
library: DOI normalised (prefix and `doi:` stripped, lowercased), then title normalised
(punctuation collapsed) plus first-author surname. On a hit it returns `{ duplicate: true, existing }`
and creates nothing. Costs up to two extra searches per create. Bypassed by `forceCreate: true`;
arrays of items skip it entirely.

**Attachment upload** (`client.ts:655-735`) implements Zotero's full four-step flow: create the
attachment item → `POST .../file` for authorization → `POST` prefix+bytes+suffix to the returned URL
→ register the upload. Honours `exists` (server already has the file, skip the transfer). Reads the
whole file into memory and md5s it; `contentType` defaults to `application/pdf`.

**Attachment download** (`client.ts:742-753`) writes bytes to the given path and returns the ETag
as `md5`. It writes wherever the agent points it, with no path checks.

**Attachment list** filters children to `itemType === "attachment"`, so child notes are invisible
to that tool. There is no notes or annotations tool; notes exist only as `zotero_item create` with
`itemType: "note"`.

**Argument coercion**: `coerceItem()` (`tools.ts:153-164`) JSON-parses the `item` parameter when a
model sends it as a string, because `Type.Any()` is not converted by pi's TypeBox layer.

**Write responses**: `parseWriteResponse()` (`client.ts:569-583`) handles Zotero's
`{ successful, unchanged, failed }` object, sorts `successful` back into submission order, and
throws when any sub-write failed.

## 7. Gaps between the code and the documentation

Facts, not opinions; each was checked against the source.

1. **"Always Allow" is not persisted.** `cfg.saveAuth` (`tools.ts:79-92`) fetches the resolved auth
   object and mutates `current.env`. `ModelRegistry` exposes only `getProviderAuth`
   (`model-registry.d.ts:35`) — there is no setter — so the mutation is discarded. The key survives
   in the in-process `localSessions` map and is gone on restart, contrary to the README's
   "saves the granted key in `~/.pi/agent/auth.json`".
2. **README overstates the Zotero version requirement.** Local *reads* work on Zotero 7+; only the
   write API needs 10+. The version check runs only in `fetchServerId`.
3. **Group libraries are unreachable from tools** except by hand-editing `ZOTERO_GROUP_ID`, and
   there is no way to discover group ids. Documented in `design/issue/local-group-libraries.md`.
4. **The version check is skipped when a Server-ID is already stored** — the write then fails with a
   bare 404. This and the README overstatement are written up in
   `design/debt/local-write-version-check.md`.

## 8. Tests

`npm test` — Node's built-in runner with `tsx`, a hand-rolled `fetch` mock (`test/_mock.ts`, 72
lines) that records calls and matches canned responses. No network, no Zotero required. 90 tests
across `client.test.ts` (paths, headers, write flow, local auth), `tools.test.ts` (each tool's
execute via a fake `ExtensionAPI`), `provider.test.ts` (login/verify/check/resolve).

One trap: `localSessions` is module-level and persists between tests, so local write tests must each
use a distinct base URL (`/api`, `/api2`, `/api-v8`) or they silently reuse a cached key.

## 9. Descope candidates

Proposals, not current behaviour. Ordered by lines removed per unit of risk. Line counts include
the matching tests.

1. **`zotero_schema` (~110 lines: `client.ts:754-823`, `tools.ts:659-696`, 5 client functions).**
   Five tools-worth of surface for something `zotero_template` already covers — a template lists the
   valid fields for an item type by construction. The schema tool only wins for exotic creator types.
   Removing it also removes the only tools that quietly require internet in local mode, along with
   `zotero_template`.
2. **Dedup on create (~70 lines, four helpers: `tools.ts:166-235`).** Real work, but it makes
   `create` do up to three round trips and give a non-obvious result (`duplicate: true`, nothing
   created). The same outcome is reachable by the agent calling `zotero_search` first, which it can
   already do. Keep if unattended batch imports matter; drop if creates are mostly agent-driven and
   interactive.
3. **Dead parameters that cost nothing but mislead** (~15 lines total, no behaviour change):
   - `cfg?: ZoteroConfig` as a trailing argument on `itemTemplate` and all five schema functions —
     no caller ever passes it, so it always falls back to the web base URL.
   - `SearchParams.since` (`client.ts:247`, `:259`) — no tool exposes it.
   - `listCollectionItems`' `q` / `qmode` options (`client.ts:396-412`) — the tool passes only
     `top` and `limit`.
   - `itemKeys` on `zotero_collection` `add`/`remove`: `normalizeMembershipItems` (`tools.ts:237-246`)
     ignores it and throws unless `items` is given, yet it is in the schema and the README.
4. **`zotero_tags` `get` is `list` with an `itemKey`** (`tools.ts:480-489`) — same client call, same
   output bar one field. Merging them removes an action and a branch.
5. **`zotero_fulltext` (~75 lines: `client.ts:824-861`, `tools.ts:697-735`).** Niche: it exists so a headless setup can index PDF text that
   the desktop client would index itself. If local mode is the normal path, this mostly never runs.
6. **Export formats.** `mods`, `coins`, `bookmarks` are pass-through strings in a union
   (`client.ts:506-515`), so they cost almost nothing to keep. Mentioned only for completeness —
   there is nothing to win here.

Not descopable without losing a headline feature: the local write-authorization machinery
(section 4), the four-step upload flow, and `summarize()` (which is what keeps tool output small
enough to be worth returning to a model).
