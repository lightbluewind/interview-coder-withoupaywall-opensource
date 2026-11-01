# Baseline Check

Date: 2026-11-01 UTC

Repository: `~/workspace/interview-coder-withoupaywall-opensource`

Branch: `codex/baseline-dead-code-cleanup`

## Environment

- Node: `v24.22.1`
- npm: `10.9.4`
- `node_modules` before install: absent

## Results

### 1. `node --version && npm --version`

- Exit code: `0`
- Result: PASS

Output:

```text
v24.22.1
10.9.4
```

### 2. `npm install`

- Exit code: `1`
- Result: FAIL
- `node_modules` after command: absent/not usable
- Tracked files changed: none

Output:

```text
npm error code EPERM
npm error errno EPERM
npm error FetchError: request to https://registry.npmjs.org/zwitch/-/zwitch-2.0.4.tgz failed, reason:
npm error     at ClientRequest.<anonymous> (/opt/hatch-image/bin/npm-package/node_modules/minipass-fetch/lib/index.js:130:14)
npm error     at ClientRequest.emit (node:events:514:28)
npm error     at emitErrorEvent (node:_http_client:114:5)
npm error     at _destroy (node:_http_client:1175:10)
npm error     at onSocketNT (node:https:726:5)
npm error     at process.processTicksAndRejections (node:internal/process/task_queues:91:21)
npm error   code: 'EPERM'
npm error   errno: 'EPERM'
npm error   type: 'system'
npm error }
npm error
npm error The operation was rejected by your operating system.
npm error It is likely you do not have the permissions to access this file as the current user
npm error
npm error If you believe this might be a permissions issue, run the command again as root/Administrator.
npm error Log files were not written due to an error writing to the directory: /home/hatch/.npm/_logs
npm error You can rerun the command with `--loglevel=verbose` to get extended logs.
```

An escalated retry was requested to distinguish a sandbox/network restriction from a repository problem, but the approval service returned `invalid_parameter_error` and did not execute it.

### 3. `npx tsc --noEmit`

- Exit code: `1`
- Result: FAIL

Output:

```text
npm error code EPERM
npm error errno EPERM
npm error FetchError: request to https://registry.npmjs.org/tsc failed, reason:
npm error     at ClientRequest.<anonymous> (/opt/hatch-image/bin/npm-package/node_modules/minipass-fetch/lib/index.js:130:14)
npm error     at ClientRequest.emit (node:events:514:28)
npm error     at emitErrorEvent (node:_http_client:114:5)
npm error     at _destroy (node:_http_client:1175:10)
npm error     at onSocketNT (node:https:726:5)
npm error     at process.processTicksAndRejections (node:internal/process/task_queues:91:21)
npm error   code: 'EPERM'
npm error   errno: 'EPERM'
npm error   type: 'system'
npm error }
npm error
npm error The operation was rejected by your operating system.
npm error It is likely you do not have the permissions to access this file as the current user
npm error
npm error If you believe this might be a permissions issue, run the command again as root/Administrator.
npm error Log files were not written due to an error writing to the directory: /home/hatch/.npm/_logs
npm error You can rerun the command with `--loglevel=verbose` to get extended logs.
```

### 4. `npm run lint`

- Exit code: `1`
- Result: FAIL

Output:

```text
> interview-coder-v1@1.0.19 lint
> npx eslint .

npm error code EPERM
npm error errno EPERM
npm error FetchError: request to https://registry.npmjs.org/eslint failed, reason:
npm error     at ClientRequest.<anonymous> (/opt/hatch-image/bin/npm-package/node_modules/minipass-fetch/lib/index.js:130:14)
npm error     at ClientRequest.emit (node:events:514:28)
npm error     at emitErrorEvent (node:_http_client:114:5)
npm error     at _destroy (node:_http_client:1175:10)
npm error     at onSocketNT (node:https:726:5)
npm error     at process.processTicksAndRejections (node:internal/process/task_queues:91:21)
npm error   code: 'EPERM'
npm error   errno: 'EPERM'
npm error   type: 'system'
npm error }
npm error
npm error The operation was rejected by your operating system.
npm error It is likely you do not have the permissions to access this file as the current user
npm error
npm error If you believe this might be a permissions issue, run the command again as root/Administrator.
npm error Log files were not written due to an error writing to the directory: /home/hatch/.npm/_logs
npm error You can rerun the command with `--loglevel=verbose` to get extended logs.
```

### 5. `npm run build`

- Exit code: `127`
- Result: FAIL
- `dist/`: absent after build attempt
- `dist-electron/`: absent after build attempt

Output:

```text
> interview-coder-v1@1.0.19 build
> cross-env NODE_ENV=production npm run clean && vite build && tsc -p tsconfig.electron.json

sh: 1: cross-env: not found
```

## Build Artifacts

- `dist/`: absent
- `dist-electron/`: absent

## Most Blocking Errors

1. `npm install` cannot complete because package downloads fail with `EPERM`; consequently, project dependencies are not installed.
2. `npx tsc --noEmit` and `npm run lint` cannot resolve local tools and fail while trying to download `tsc`/`eslint`.
3. `npm run build` cannot start because `cross-env` is not installed.

## Baseline Verdict

FAIL. Part 2 dead-code cleanup was not started.

---

## Cloud re-run (Muse, outside the Codex sandbox) — 2026-11-01 19:31 SGT

The Codex-sandbox failures above were environment-only: on the same cloud
computer, run by Muse outside the Codex sandbox, the npm registry is reachable
via the egress proxy and the install succeeds.

- `npm install`: PASS, exit 0, 839 packages added in 50s
  (log: `~/workspace/codex-tasks/npm-install-muse.log`)
- `npx tsc --noEmit` (root `tsconfig.json`, `strict: true`): FAIL, exit 2,
  48 errors — 14 in `electron/main.ts`, 14 in `electron/ProcessingHelper.ts`,
  10 in `src/_pages/Debug.tsx`, 4 in `src/_pages/Solutions.tsx`,
  2 in `electron/ScreenshotHelper.ts`, 2 in `src/components/UpdateNotification.tsx`,
  1 in `src/components/Header/Header.tsx`, 1 in `electron/autoUpdater.ts`.
  Types: implicit `any` (TS7006/TS7034/TS7005), possible-null (TS18047/TS18049),
  type mismatches (TS2322/TS2345), unknown error (TS18046).
  None of the 48 errors is in a dead-code candidate
  (`renderer/`, `src/_pages/SubscribePage.tsx`, `src/lib/supabase.ts`).
  (log: `~/workspace/codex-tasks/baseline-muse.log`)
- `npm run lint` (`npx eslint .`): FAIL, exit 1, 106 errors, 0 warnings.
  Includes parsing errors on non-code files (`*.md`, `*.json`, `*.css`,
  including this `BASELINE.md` itself) because lint runs on `.` with no
  ignore list, plus `no-explicit-any` / unused-variable errors in core files.
- `npm run build`: PASS, exit 0. Vite built the renderer (2769 modules),
  electron main (607 modules) and preload (1 module);
  `dist/` (3.3M) and `dist-electron/` (4.9M) were produced.
  The build's own `tsc -p tsconfig.electron.json` passes separately
  (exit 0) because that config sets `strict: false`,
  `noImplicitAny: false`, `strictNullChecks: false`, unlike the root config.

## Cloud baseline verdict

Mixed: install and build pass, so the project does build on the cloud
computer. Strict `tsc --noEmit` and `lint` fail on pre-existing code/config
issues unrelated to the dead-code candidates. Part 2 was not started yet
pending a decision on whether to proceed with cleanup measured by
before/after error counts rather than requiring a zero-error baseline.
