# Dead-Code Cleanup Report

Date: 2026-11-01 (UTC)
Repository: `~/workspace/interview-coder-withoupaywall-opensource`
Branch: `codex/baseline-dead-code-cleanup`
Status: **all changes are uncommitted and unstaged; nothing was pushed, no PR created.**

Scope was dead-code cleanup only, measured by before/after error counts
(counts must not increase, build must still pass). The 48 pre-existing strict
`tsc` errors and the pre-existing lint errors were deliberately **not** fixed.

---

## 1. Before / after measurements

All runs from the repo root. `dist/` and `dist-electron/` are gitignored build
artifacts; because `npm run lint` runs `eslint .` with no ignore list, it also
lints those generated bundles. Counts are therefore reported in both states so
the comparison is apples-to-apples.

| Check | Before | After | Delta |
|---|---|---|---|
| `npx tsc --noEmit` | exit 2, 48 errors | exit 2, 48 errors | 0 (same 8 files, same counts) |
| `npm run lint` (no `dist/`, report file excluded) | exit 1, 106 errors | exit 1, 97 errors | **-9** |
| `npm run lint` (no `dist/`, incl. this `CLEANUP.md`) | n/a | exit 1, 98 errors | 97 code + 1 for this file's own markdown parse error |
| `npm run lint` (with `dist/`, incl. `CLEANUP.md`) | exit 1, 1041 errors | exit 1, 1033 errors | -8 |
| `npm run build` | PASS, exit 0 (baseline) | PASS, exit 0 | unchanged |

The +1 in the last two rows is the eslint config's pre-existing behaviour of
parsing markdown as if it were JavaScript (`BASELINE.md` already produced the
same "Parsing error" error in the 106 baseline); `CLEANUP.md` is a review-only
document that will be removed or ignored before any commit. The code-level
delta is -9 in both artifact states (1041-9=1032, +1 report file = 1033).

`tsc` breakdown is identical before and after — the deletions touched none of
these files:

```
14 electron/main.ts            14 electron/ProcessingHelper.ts
10 src/_pages/Debug.tsx         4 src/_pages/Solutions.tsx
 2 src/components/UpdateNotification.tsx
 2 electron/ScreenshotHelper.ts 1 src/components/Header/Header.tsx
 1 electron/autoUpdater.ts
```

The -9 lint errors are exactly the files that were deleted:

```
renderer/README.md            1 -> 0
renderer/package.json         1 -> 0
renderer/public/manifest.json 1 -> 0
renderer/src/App.css          1 -> 0
renderer/src/App.test.tsx     2 -> 0
renderer/src/index.css        1 -> 0
renderer/tsconfig.json        1 -> 0
src/types/global.d.ts         3 -> 2   (removed `supabase: any` field)
```

Build result after cleanup (`npm run build`, exit 0):

- Vite renderer bundle: 2768 modules, `dist/` = 3.3M, `dist/index.html` present
- Electron main: 607 modules, preload: 1 module, `dist-electron/` = 4.9M,
  `dist-electron/main.js` and `dist-electron/preload.js` present
- The build's own `tsc -p tsconfig.electron.json` (non-strict) passed as part
  of the same command.

---

## 2. Deleted

### 2.1 `renderer/` — entire directory (19 tracked files)

Legacy Create React App project (`renderer/package.json`, `renderer/src/*`,
CRA public assets). Nothing in the active app references it. Verified:

- `vite.config.ts`: single Vite root (repo root), entries `electron/main.ts`
  and `electron/preload.ts`, alias `@` -> `./src`. No `renderer` path.
- `tsconfig.json` include: `["electron/**/*", "src/**/*"]` — `renderer/` is not
  compiled by the root strict config.
- `tsconfig.node.json` include: `["vite.config.ts"]`; `electron/tsconfig.json`
  include: `["*.ts", "../src/types/electron.d.ts"]` — neither reaches it.
- `package.json`: no `workspaces`, no script references `renderer`.
- `index.html` / `src/main.tsx`: root renderer entry only.
- `electron/main.ts`, `electron/preload.ts`, all of `src/`: no import path
  containing `renderer`.
- `.github/workflows/ci.yml` only runs `npm run clean|lint|build|test`; no
  `renderer` step. `cleanup-for-github.sh/.bat` do not mention it.
- `README.md`, `CONTRIBUTING.md`, `CHANGES.md`: no mention of `renderer/`.
- No `renderer/node_modules`, so nothing depended on its own install tree.

Remaining textual matches for "renderer" in the repo are unrelated: comments
about the Electron *renderer process* (`electron/autoUpdater.ts`,
`ipcHandlers.ts`, `shortcuts.ts`) and the `vite-plugin-electron-renderer`
package name.

### 2.2 `src/_pages/SubscribePage.tsx`

Dead page. Verified with a repo-wide grep for `SubscribePage`: the only hits
were its own `interface SubscribePageProps` / `export default function
SubscribePage` and `BASELINE.md`. No importer in `src/` or `electron/`. It was
the sole consumer of `window.electronAPI.openSubscriptionPortal` and of the
`User` type from `@supabase/supabase-js`.

### 2.3 `src/lib/supabase.ts`

Supabase shim whose only two importers were `SubscribePage.tsx` (deleted above)
and `SolutionCommands.tsx` (replaced below). After both were resolved a
repo-wide grep for `lib/supabase` / `@supabase` returns nothing, so the file was
removed.

### 2.4 `env.d.ts` (repo root) and `src/vite-env.d.ts`

Both files contained **only** `VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY`
declarations (plus a `vite/client` triple-slash reference). Verified dead:

- `grep -rn "VITE_SUPABASE"` now matches nothing outside history docs.
- `grep -rn "import\.meta\.env"` matches nothing in `src/` or `electron/`, i.e.
  no code reads those variables.
- The root file was not in any `tsconfig` include; `src/vite-env.d.ts` only
  duplicated ambient env typing, and `tsconfig.json` already sets
  `"types": ["vite/client"]`.
- No triple-slash reference to either file anywhere.

Removing these two files is a direct consequence of removing Supabase; it is
called out separately here because they were not listed as candidates.

---

## 3. Changed

### 3.1 `src/components/Solutions/SolutionCommands.tsx` — sign-out rewritten

Removed `import { supabase } from "../../lib/supabase"`. `handleSignOut` is now
a local-only cleanup:

```ts
localStorage.clear()
sessionStorage.clear()
await window.electronAPI.updateConfig({ apiKey: "" })
window.location.reload()
```

Why this is safe, verified in source:

- `electron/preload.ts` exposes `updateConfig` -> `ipcRenderer.invoke("update-config", ...)`.
- `electron/ipcHandlers.ts` registers `ipcMain.handle("update-config", ...)` -> `configHelper.updateConfig(updates)`.
- `electron/ConfigHelper.ts` `updateConfig()` merges the partial into the stored
  config and persists it. With `apiKey: ""` the provider auto-detection branch
  is skipped (empty string is falsy), so `apiProvider`, models, language and
  opacity are preserved; only the key is blanked.
- `electron/ProcessingHelper.ts` listens for `config-updated` and calls
  `initializeAIClient()`, which sees an empty key and nulls the clients with a
  warning instead of throwing — so no crash between clear and reload.
- `window.location.reload()` matches the existing pattern in `src/App.tsx`
  (`handleApiKeySave`). After reload `checkApiKey()` is false, so the app falls
  back to `WelcomeScreen` — the intended "logged out" state for a build with no
  auth server.

`preload.ts`'s `clearStore()` was **not** used: it invokes `"clear-store"`, for
which no `ipcMain.handle` exists (pre-existing bug, left untouched).

### 3.2 `electron/main.ts` — `supabase.co` removed from the allowed-hosts check

```diff
-const allowedHosts = ["google.com", "supabase.co"];
-if (allowedHosts.includes(hostname) || hostname.endsWith(".google.com") || hostname.endsWith(".supabase.co")) {
+const allowedHosts = ["google.com"];
+if (allowedHosts.includes(hostname) || hostname.endsWith(".google.com")) {
```

Made only after confirming zero live Supabase use remains: no `@supabase`
import, no `supabase.ts`, no auth callback, no Supabase URL anywhere in code.
`google.com` handling is unchanged.

### 3.3 `src/types/global.d.ts` — `supabase: any` field removed

Only valid while `src/lib/supabase.ts` existed; removed together with it.
`electron`, `electronAPI`, `__CREDITS__`, `__LANGUAGE__`, `__IS_INITIALIZED__`,
`__AUTH_TOKEN__` untouched.

### 3.4 `package.json` / `package-lock.json` — `@supabase/supabase-js` removed

Nothing imports it any more (`grep -rn "@supabase" src electron` -> none), so
the dependency entry was dropped.

`node_modules` was **not** reinstalled or deleted. The lockfile was updated in
place with:

```
npm install --package-lock-only --ignore-scripts
```

Exit 0, "up to date in 2s". Lock diff is limited to the `@supabase/*` entries:
`2 insertions(+), 90 deletions(-)`. `package.json`, publish owner, README clone
URL, model lists and scripts are otherwise untouched. `node_modules/@supabase`
still physically exists on disk but is no longer a declared dependency; it
drops out on the next clean install.

### 3.5 `src/env.d.ts` — Supabase env fields removed

Removed `VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY` from `ImportMetaEnv`
(same reasoning as 2.4). The `ElectronAPI` / `Window` declarations in that file
were left as-is.

---

## 4. Deliberately left in place

| Item | Reason |
|---|---|
| `src/_pages/SubscribedApp.tsx` | Despite the legacy name it is the live app shell rendered by `src/App.tsx` when an API key exists. Rename is a separate task. |
| `electron/ProcessingHelper.ts`, `ScreenshotHelper.ts`, `ConfigHelper.ts`, `shortcuts.ts`, `ipcHandlers.ts`, `autoUpdater.ts` | Core pipeline, out of scope. The only edit to that surface is the one-line host-list change in `main.ts`. |
| `window.electronAPI.openSubscriptionPortal` (`preload.ts`), `onSubscriptionUpdated`, `onSubscriptionPortalClosed` and their type declarations in `src/env.d.ts` / `src/types/electron.d.ts` | Now uncalled (their only caller, `SubscribePage`, was deleted), but they are part of the Electron API surface and `open-subscription-portal` has no main-process handler anyway. Removing them is an API change, not dead-code tidy-up; left as follow-up. |
| `preload.ts` `clearStore` / `"clear-store"` | Pre-existing: the invoke has no registered `ipcMain` handler, so it rejects at runtime. Pre-existing bug, unrelated to this cleanup; not used by the new sign-out path. |
| `__AUTH_TOKEN__` in `src/types/global.d.ts`, `src/env.d.ts`, `src/types/electron.d.ts` | Auth-era leftover with no reader, but it is not Supabase-specific and removing it is a type-surface change; flagged as follow-up. |
| Supabase mentions in `CHANGES.md` and the historical comment at `electron/main.ts:192` | History/documentation, not code. |
| The 48 pre-existing `tsc` errors and remaining 97 lint errors | Out of scope per the task; measured only to prove no regression. |
| Root `tsconfig.json` / `eslint.config.mjs` ignore-list gaps | The lint config has no `dist`/`dist-electron` ignore, which is why lint counts swing by ~935 between artifact states. Fixing config is a separate task. |

---

## 5. Verification performed

Commands, in order, from the repo root after the edits:

1. `npx tsc --noEmit` -> exit 2, 48 errors (identical file breakdown to baseline)
2. `npx eslint .` -> exit 1, 97 errors (baseline 106; no new errors, deleted
   files account for all -9). Re-run with build artifacts present: 1032 vs 1041.
3. `npm run build` -> exit 0; `dist/` (3.3M) and `dist-electron/` (4.9M)
   produced, `dist/index.html`, `dist-electron/main.js`,
   `dist-electron/preload.js` all present.

Additional checks:

- `grep -rn "lib/supabase\|SubscribePage\|@supabase" src electron` -> no matches.
- `grep -c 'apiKey: ""'` in `SolutionCommands.tsx` -> 1 (the intended literal).
- `grep -o "update-config" dist-electron/*.js` -> present in `preload.js`,
  `ipcHandlers.js` and the bundled main chunk, i.e. the new sign-out path
  survives bundling end to end.
- No `npm install`, no `node_modules` deletion, no `git add`, no commit.

---

## 6. Diff summary

```
29 files changed, 10 insertions(+), 653 deletions(-)
```

Deleted (23): `renderer/` (19 files), `src/_pages/SubscribePage.tsx`,
`src/lib/supabase.ts`, `env.d.ts`, `src/vite-env.d.ts`.
Modified (6): `electron/main.ts`, `src/components/Solutions/SolutionCommands.tsx`,
`src/types/global.d.ts`, `src/env.d.ts`, `package.json`, `package-lock.json`.

`BASELINE.md` and this `CLEANUP.md` are untracked working files (review aids),
not part of the change set.