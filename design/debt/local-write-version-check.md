# Review: `f81acb1` — enforce Zotero 10+ for the local write API

Reviewed 2026-10-12 against `f81acb1` (the HEAD commit; no uncommitted code changes).
Status: **open** — nothing in this list has been fixed yet.

What was read: the diff, the write-auth path around it (`client.ts:121-205`), the `/login` flow
in `provider.ts:145-170`, how `tools.ts:76` feeds the stored Server-ID back in, and the local-mode
tests. Full suite passes: 90/90.

**What this change does:** When pi needs to write to the Zotero desktop app, it first asks that app
for its Server-ID. This change makes that step also read the app's version header and refuse with a
clear message if Zotero is older than 10, because the local write API only exists in Zotero 10+.
The README now states the 10+ requirement, and a test covers the refusal.

Assumed load: one person, one desktop Zotero, a handful of writes per session. Nothing here is
load-sensitive.

## Must fix

1. **The version check is skipped in the normal case, so the user still gets the confusing 404**
   (`client.ts:70-90`, `client.ts:128-134`)
   - **What this is:** `fetchServerId` is the only place the version is read. It runs only when pi
     does not already know the Server-ID. But `/login zotero` stores the Server-ID in `auth.json`
     (`provider.ts:153-164`), and `resolveConfig` passes it back in (`tools.ts:76`), so on a
     configured machine that call never happens.
   - **Problem:** With a stored Server-ID and an old Zotero, pi skips straight to
     `POST /local/authorize`, which does not exist before Zotero 10. Reproduced against a mocked
     Zotero 8: the error is `Failed to authorize local write (404): Not Found` — exactly the message
     this commit set out to replace. The only fetch made was the authorize call; the version was
     never looked at. This bites whenever a pre-10 Zotero sends the `Zotero-Server-ID` header, which
     is what the new test assumes (it returns that header with version `8.0.4`).
   - **Fix:** Translate the 404 where it happens, in `authorizeLocalWrite` (`client.ts:97-118`),
     next to the existing 403 and 429 branches:
     ```ts
     if (res.status === 404) {
       throw new ZoteroError("Local Zotero write API requires Zotero 10+. Please update Zotero or use Web API mode.", res.status, "");
     }
     ```
     Three lines, no extra request, and it works whether or not the version header is present. Keeps
     the `fetchServerId` check as the earlier, more specific message.
   - **If we skip it:** The one user group this commit targets — people on old Zotero — mostly still
     sees the raw 404 and has no idea they need to upgrade.

## Should fix

2. **The new test covers the path that rarely runs** (`test/client.test.ts:597-616`)
   - **What this is:** The test starts from a config with no Server-ID, so `fetchServerId` runs and
     the version check fires.
   - **Problem:** The configured-machine path from finding 1 (`serverId` already set) has no test,
     so the gap went unnoticed.
   - **Fix:** Add one test alongside it: same mock, but call
     `createItems({ ...LOCAL_CFG, baseUrl: ".../api-v8b", serverId: "SRV8" }, …)` with
     `/local/authorize` returning 404, and assert the message matches `/Zotero 10\+/`. It fails today
     and passes once finding 1 is fixed.
   - **If we skip it:** The same hole can reopen the next time the auth flow is touched.

3. **Tests silently share write keys unless each one invents a new base URL**
   (`client.ts:66` and `test/client.test.ts:534-616`)
   - **What this is:** `localSessions` is a module-level cache keyed by base URL. It survives between
     tests, and the first local test stores a remembered write key under `http://localhost:23119/api`.
   - **Problem:** The last two tests only work because they use made-up base URLs (`/api2`,
     `/api-v8`). A future test that reuses `LOCAL_CFG` unchanged would reuse the cached key, skip
     authorization entirely, and pass without testing anything.
   - **Fix:** One comment above `LOCAL_CFG`:
     `// localSessions is cached per baseUrl across tests — give each write test its own baseUrl.`
     Cheaper than a test-only reset export.
   - **If we skip it:** A later test quietly tests nothing, which is worse than failing.

## Nice to have

4. **The version parse is five lines longer than needed, and one branch is unreachable**
   (`client.ts:72-80`)
   - **What this is:** It splits the header on `.`, defaults to `"0"`, parses, then guards with
     `majorVersion &&`.
   - **Problem:** `parseInt` already stops at the first dot, so `parseInt("8.0.4", 10)` is `8`. A
     missing header gives `NaN`, and `NaN < 10` is false, so the guard is redundant too. The
     `|| "pre-10"` fallback in the message can never print: if the header is empty the code does not
     reach the throw.
   - **Fix:**
     ```ts
     const zoteroVersion = res.headers.get("x-zotero-version") || "";
     if (parseInt(zoteroVersion, 10) < 10) {
       throw new ZoteroError(`Local Zotero write API requires Zotero 10+ (detected ${zoteroVersion}). Please update Zotero or use Web API mode.`, 400, "");
     }
     ```
     Same behaviour in every case, including a header like `7.0.0-beta`.
   - **If we skip it:** Nothing breaks; it is five more lines to read and a misleading dead string.

5. **The README now overstates the requirement** (`README.md:17`)
   - **What this is:** It says local mode needs Zotero desktop 10+.
   - **Problem:** Only writes need 10+. Reading works on Zotero 7/8, and the extension picks local
     mode automatically whenever port 23119 answers. A Zotero 8 user who only searches their library
     is told to upgrade for no reason.
   - **Fix:** "Zotero desktop running locally … Reading works on Zotero 7+; creating or editing items
     requires Zotero 10+."
   - **If we skip it:** Some users switch to a web API key they do not need.

Verdict: fix 1 and 2 first.
Lean: -5 lines possible.
Not checked: a real pre-10 Zotero desktop. Whether finding 1 bites every old-Zotero user or only
some depends on whether pre-10 sends the `Zotero-Server-ID` header; the new test assumes it does.
The suggested 404 fix is correct either way.
