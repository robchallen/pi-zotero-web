# pi-zotero-web

Zotero integration for [pi](https://pi.dev) supporting both the **Zotero Local API** (desktop app) and the **Zotero Web API**.

- **Dual-mode support** — automatically defaults to the high-performance offline **Local API** (`http://localhost:23119/api/`) when Zotero desktop is running, falling back to the remote **Web API** (`https://api.zotero.org`).
- **Interactive local write authorization** — write operations on the Local API interactively trigger Zotero's desktop approval modal ("Allow" / "Always Allow" / "Deny") with automatic key persistence. Reads need no credentials.
- **CRUD on paper metadata + PDFs** — create, read, update, delete items and attachments; upload/download PDF files.
- **Library search** — keyword, title/author/year, and full-text (`qmode=everything`) search.
- **Tags & collections** — manage tags, list/create/rename/delete collections, and move items in/out of collections.
- **Citation export** — export items as BibTeX, BibLaTeX, CSL JSON, RIS, CSV, MODS, COinS, bookmarks, or a formatted bibliography.
- **Schema & full-text** — read the item-type/field/creator schema, and get/set extracted full-text content for attachments.
- **Credential management via `/login zotero`** — autodetects desktop Zotero, allows picking Local vs Web mode, and persists credentials in `~/.pi/agent/auth.json`.

## Requirements

Either:
- **Local Mode (Recommended):** Zotero desktop 10+ running locally with **Settings → Advanced → "Allow other applications on this computer to communicate with Zotero"** enabled. (Local write API and interactive authorization require Zotero 10+). No web API key needed.
- **Web API Mode:** A Zotero account with an API key created at <https://www.zotero.org/settings/keys> with **Allow library access** and **Allow file access** (plus write permissions if modifying items).

## Install

Install the published npm package (writes to `~/.pi/agent/settings.json`):

```bash
pi install npm:pi-zotero-web
# or pin a version:
pi install npm:pi-zotero-web@0.2.2
```

Add `-l` to install into project settings (`.pi/settings.json`) instead of user settings.

Alternatively, test it from a checkout without installing:

```bash
pi -e ./index.ts
```

## Configure

Run `/login zotero`. If Zotero desktop is running, it prompts you to select between:
1. **Local Zotero Desktop (Automatic / no web API key needed)**
2. **Zotero Web API Key**

### 1. Local Mode
When Local Mode is active:
- Read operations require **no configuration or API key**.
- Write operations (create, update, delete, upload) trigger a native Zotero confirmation prompt on your desktop.
- Selecting **"Always Allow"** saves the granted key in `~/.pi/agent/auth.json` keyed by the local database's `Zotero-Server-ID`, avoiding prompts on future operations.
- Selecting **"Allow"** grants a single-use key consumed on that write.

### 2. Web API Mode
If choosing Web API Mode, paste your Zotero Web API key. It is verified against `/keys/current` and stored with your numeric user id in `~/.pi/agent/auth.json`:

```jsonc
{
  "zotero": {
    "type": "api_key",
    "key": "<your-key>",
    "env": {
      "ZOTERO_MODE": "web",
      "ZOTERO_USER_ID": "12345"
    }
  }
}
```

To force a specific mode or group library via environment configuration in `auth.json`:
- `"ZOTERO_MODE": "local"` or `"web"`
- `"ZOTERO_GROUP_ID": "<group-id>"`
- `"ZOTERO_BASE_URL": "http://localhost:23119/api"`

Remove stored credentials anytime with `/logout zotero`.

## Tools

All tools are registered for the agent to call automatically. You can also prompt for them directly (e.g. *"Search my Zotero library for papers on diffusion policies"*).

| Tool | Actions / params | Description |
|------|------------------|-------------|
| `zotero_search` | `q`, `qmode`, `itemType`, `collectionKey`, `tag`, `limit`, `top` | Search the library. `qmode=everything` includes full text. Returns key/version/metadata. |
| `zotero_item` | `action=get\|create\|update\|delete`, `itemKey`, `version`, `item`, `forceCreate` | CRUD on an item. Use the `version` returned by search/get for update/delete. On `create`, the library is first searched for an existing item with the same DOI (or title + first author); a match is returned with `duplicate:true` instead of creating. Pass `forceCreate:true` to bypass. |
| `zotero_template` | `itemType`, `linkMode` | Fetch an item template to build valid create/update payloads. |
| `zotero_attachment` | `action=list\|upload\|download\|delete`, `itemKey`/`parentKey`, `filePath`, `version`, `title`, `contentType` | Manage PDF/attachment files. Upload reads a local file; download writes to a local path. |
| `zotero_tags` | `action=list\|get\|set`, `itemKey`, `version`, `tags`, `limit` | List library tags or tags on an item, or replace an item's full tags array (`tags: [{tag, type:0\|1}]`, type 0=manual, 1=automatic). |
| `zotero_collection` | `action=list\|get\|items\|create\|rename\|delete\|add\|remove`, `collectionKey`, `parentKey`, `name`, `version`, `itemKeys`/`items`, `top`, `limit` | Manage collections (folders): list/get/create/rename/delete, list items in a collection, and add/remove items to/from a collection (requires each item's current `version` + `collections`). |
| `zotero_export` | `format`, `itemKeys`, `collectionKey` | Export items as BibTeX, BibLaTeX, CSL JSON, RIS, CSV, MODS, COinS, bookmarks, or a formatted bibliography (`bib`). Select by `itemKeys` or a `collectionKey`. |
| `zotero_schema` | `action=itemTypes\|itemFields\|itemTypeFields\|creatorTypes\|creatorFields`, `itemType` | Read-only access to Zotero's item-type schema (public endpoints, no auth) so the agent can build valid create payloads for exotic types. |
| `zotero_fulltext` | `action=get\|set`, `itemKey`, `content`, `indexedChars`/`totalChars` or `indexedPages`/`totalPages` | Get or set extracted full-text content for an attachment, enabling `qmode=everything` search without the desktop client. |

### Typical workflows

Create a paper from a DOI/metadata:

1. `zotero_template` (or `zotero_schema` for exotic types) → base object.
2. Fill `title`, `creators`, `abstractNote`, `DOI`, `date`, etc. (the agent can fetch these from the web).
3. `zotero_item` with `action=create`, passing the filled item.

> **Duplicate detection.** `create` checks the library for an existing item matching the new one (by DOI, then by normalized title + first author) and returns the existing entry with `duplicate:true` instead of creating a second copy. Pass `forceCreate:true` to always create. Batches (an array of items) always create without dedup to avoid ambiguous matches.

Attach a PDF to an existing item:

1. `zotero_attachment` with `action=upload`, `parentKey=<itemKey>`, `filePath=/path/to/paper.pdf`.

Download a PDF:

1. `zotero_attachment` with `action=download`, `itemKey=<attachmentKey>`, `filePath=./paper.pdf`.

Organize and export:

1. `zotero_collection` `action=create` → new collection; `action=add` to file items into it (pass each item's current `version` + `collections`).
2. `zotero_export` with `format=bibtex`, `collectionKey=<key>` → BibTeX for the whole collection.

Make an uploaded PDF searchable headlessly:

1. Extract text locally (e.g. via a PDF tool), then `zotero_fulltext` `action=set` with `content` + `indexedChars`/`totalChars` (text) or `indexedPages`/`totalPages` (PDF). Now `zotero_search` with `qmode=everything` will find it.

## Testing

Unit tests use Node's built-in test runner with a mocked `fetch` (no real network):

```bash
npm test
```

The pi host packages (`@earendil-works/pi-coding-agent`, `@earendil-works/pi-ai`, `typebox`) are provided by pi at runtime. For tests, they're symlinked into `node_modules/@earendil-works*` from the global pi install. If you move this checkout, re-create those symlinks (or `npm install` a local pi-coding-agent).

## How it works

- `provider.ts` registers a native pi-ai `Provider` named `zotero` declaring **no LLM models**. `/login zotero` runs provider authentication, detecting local vs web environments.
- Tools read configuration via `resolveConfig(ctx)`:
  - If local Zotero is responding on `localhost:23119`, it defaults to the **Local API** (`http://localhost:23119/api/`). Reads are sent without credentials.
  - On write operations, `client.ts` automatically requests authorization via `POST /api/local/authorize` with `Zotero-Server-ID`. If the user selects "Always Allow", the key is remembered; if "Allow" was chosen, the key is single-use.
  - If local Zotero is not running or if `ZOTERO_MODE="web"`, it calls the Zotero Web API (`https://api.zotero.org`) authenticated by `Zotero-API-Key`.
- File upload implements Zotero's full 4-step flow (create attachment item → upload authorization → upload bytes → register upload), targeting local upload endpoints in Local Mode or S3 in Web API Mode.

## License

MIT