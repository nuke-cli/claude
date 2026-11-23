# react-ts: dependency audit decisions

A running log of the decisions made while fixing `npm audit` findings in [`nuke-cli/react-ts`](https://github.com/nuke-cli/react-ts). Add a new entry at the bottom whenever another decision is made.

**Branch:** `audit-packages`
**PR:** [nuke-cli/react-ts#4](https://github.com/nuke-cli/react-ts/pull/4)

## Current status

| Severity | Start (2026-11-23) | Now |
|---|---|---|
| Critical | 4 | **0** |
| High | 14 | **0** |
| Moderate | 17 | 10 |
| Low | 1 | **0** |
| **Total** | **36** | **10** |

The 10 remaining findings are all the Jest chain (item A). See [Open items](#open-items).

## Ground rules

- **No breaking changes without an explicit decision.** Fixes use `npm update <pkg>` to stay within the version ranges `package.json` already allows. `npm audit fix --force` is not used.
- **Fix by severity, one step at a time.** Critical first, then high, then moderate, each verified separately.
- **Verify every step** with `npm test`, `npm run build` and, when dev-server packages change, `npm run dev`.
- **Commit only what is staged.** Lockfile changes are committed separately from unrelated work in progress.

## Decision log

### 1. Critical: fixed (2026-11-23)

**Decision:** Update the 4 affected transitive dependencies within their existing ranges.

| Package | Before | After | Issue |
|---|---|---|---|
| `handlebars` | 4.7.9 | 4.7.10 | JavaScript injection through crafted templates |
| `proxy-addr` | 2.0.7 | 2.0.8 | IP spoofing (dev server) |
| `shell-quote` | 1.8.3 | 1.12.0 | Unescaped newlines; quadratic-time DoS |
| `websocket-driver` | 0.7.4 | 0.7.5 | Compression limit bypass; message corruption (dev server) |

**Result:** 0 critical. Tests and build pass.
**Commit:** `fix(deps): patch critical audit vulnerabilities`

### 2. High: fixed (2026-11-23)

**Decision:** Update all 14 affected packages within their existing ranges. All had a non-breaking fix.

Packages: `@babel/plugin-transform-modules-systemjs`, `body-parser`, `brace-expansion`, `browserslist`, `compression`, `fast-uri`, `follow-redirects`, `immutable`, `js-yaml`, `launch-editor`, `nanoid`, `postcss`, `source-map-js`, `ws`.

**Result:** 0 high. Tests and build pass.
**Commit:** `ffc5a29 fix(deps): patch high severity audit vulnerabilities`

### 3. Lockfile review: approved with notes (2026-11-23)

Reviewed the 47 lockfile changes in PR #4 against `main`.

**Passed:**
- Every package resolves from `registry.npmjs.org` and has an integrity hash.
- `npm audit signatures`: all 1,074 packages have verified registry signatures.
- No changed package adds an install script.
- No semver-major bumps. The largest is `shell-quote` 1.8.3 → 1.12.0 (minor).
- All changes are dev dependencies. `react` and `react-dom` are untouched.
- The new `@jest/react-is-18` and `@jest/react-is-19` entries are npm aliases of the official `react-is`, pulled in by `@types/jest` → `pretty-format`.

**Accepted risk: very new releases.** Seven updated packages were published less than two weeks before the review:

| Package | Published | Age at review |
|---|---|---|
| `@babel/*` 7.29.15 | 2026-11-20 | 3 days |
| `source-map-js` 1.2.4 | 2026-11-19 | 4 days |
| `browserslist` 4.29.6 | 2026-11-16 | 7 days |
| `immutable` 5.2.1 | 2026-11-13 | 10 days |
| `postcss` 8.5.33 | 2026-11-13 | 10 days |
| `ws` 8.23.0 | 2026-11-12 | 11 days |
| `brace-expansion` 1.1.22 / 2.1.8 | 2026-11-11 | 12 days |

They are signed and come from their usual maintainers. If a minimum release-age policy is adopted, re-check these before merging, or pin them one patch lower.

### 4. Moderate: fixed where non-breaking (2026-11-23)

**Decision:** Fix every moderate finding that can be fixed within existing ranges. Stop at findings that need a major upgrade.

| Package | After | Notes |
|---|---|---|
| `http-proxy-middleware` | 2.0.12 | Host-header routing bypass |
| `postcss-selector-parser` | 7.2.1 | Recursion / quadratic-complexity DoS |
| `express` | 4.22.3 | Brings patched `qs@6.16.2` |
| `qs` | 6.16.2 | Now patched under both `body-parser` and `express` (previously `express` still had 6.14.x) |
| `webpack-cli` | 7.3.1 | Was flagged through `webpack-dev-server` |

**Result:** total 17 → 14 (13 moderate, 1 low). `npm test` passes, `npm run build` compiles, and the dev server starts and returns HTTP 200.
**Commit:** pending. Only `package-lock.json` changed.

### 5. `webpack-dev-server` 5 → 6: upgraded (2026-11-23)

**Decision:** Accept the semver-major upgrade to `webpack-dev-server@^6.1.0` (resolves open item B). It is a step forward rather than a downgrade, and it is the only fix for the `sockjs` → `uuid` finding.

**What changed:**
- `package.json`: `"webpack-dev-server": "^5.2.3"` → `"^6.1.0"`.
- `sockjs` and `uuid` were removed from the dependency tree. v6 drops SockJS and uses the native `ws` WebSocket server.
- No changes to `webpack.config.js` were needed.

**Compatibility review** (against the [v6 migration guide](https://github.com/webpack/webpack-dev-server/blob/main/migration-v6.md)):
- The project's `devServer` options (`static`, `hot`, `historyApiFallback`, `port`, `open`) are not affected by any documented v6 breaking change.
- The project doesn't use any of the v6 breaking areas: `proxy` / `bypass`, `setupMiddlewares` / custom Express middleware (v6 moves to Express 5), `server: 'spdy'`, `webSocketServer: 'sockjs'`, or `Server.internalIP`.
- Peer requirements are met: `webpack@5.106.2` (needs `^5.101.0`) and `webpack-cli@7.3.1` (supports `^5.0.0 || ^6.0.0`).
- **New requirement: Node.js >= 22.15.0.** Local development uses Node 24.21.0. Anyone running `npm run dev` on an older Node must upgrade. Consider adding `"engines": { "node": ">=22.15.0" }` to `package.json`.
- v6 removed the dev server's own CLI flags. The `dev` script (`webpack serve --mode development`) still works because `--mode` is a `webpack-cli` flag.

**Verification:**
- `npm run dev`: compiles; `/` returns 200; an unknown route (`/some/route`) returns 200 with `index.html` (`historyApiFallback` works); the `/ws` endpoint accepts the WebSocket upgrade (HTTP 101) for hot reload.
- `npm test`: 1/1 passing.
- `npm run build`: compiles successfully.

**Result:** total 14 → 11 (10 moderate, 1 low). Only the Jest chain and `@babel/core` remain.
**Commit:** pending. `package.json` and `package-lock.json` changed.

### 6. Low: fixed (2026-11-23)

**Decision:** Update `@babel/core` within its existing range (resolves open item C).

| Package | Before | After | Issue |
|---|---|---|---|
| `@babel/core` | 7.29.0 | 7.29.13 | Arbitrary file read via a `sourceMappingURL` comment ([GHSA-4x5r-pxfx-6jf8](https://github.com/advisories/GHSA-4x5r-pxfx-6jf8)) |

`7.29.13` was published on 2026-11-04 (19 days before the fix), so it falls outside the very-new-release concern in decision 3.

**Result:** total 11 → 10 (10 moderate, 0 low). `npm test` passes and `npm run build` compiles.
**Commit:** pending. Only `package-lock.json` changed for this step.

**Re-check (2026-11-23):** `jest@30.5.2` is the latest release, but its `jest-config` still depends on `glob@^13.0.6`, which is inside the vulnerable range. Upgrading Jest doesn't resolve item A yet.

## Open items

### A. Jest 30 → `glob` / `minimatch` (10 moderate): deferred

**Decision:** Do not downgrade Jest. Wait for an upstream Jest 30.x release that updates `glob`. Accepted as a known risk.

**Identified:** 2026-11-23

#### Dependency chain

```
jest@30.3.0
└─ @jest/core, jest-cli, jest-config, jest-runner, jest-runtime, jest-circus, @jest/reporters
   └─ glob@10.5.0        GHSA-3g3m-2fjr-6pw4 (AbortSignal cancellation bypass)
      └─ minimatch@9.0.9 GHSA-g3hv-75wg-m249 (ReDoS via globparts amplification)
```

| Package | Vulnerable range | Advisory |
|---|---|---|
| `glob` | 9.0.0 – 14.0.1 | [GHSA-3g3m-2fjr-6pw4](https://github.com/advisories/GHSA-3g3m-2fjr-6pw4): AbortSignal cancellation bypass allows post-timeout filesystem traversal |
| `minimatch` | 7.0.0 – 11.0.0 | [GHSA-g3hv-75wg-m249](https://github.com/advisories/GHSA-g3hv-75wg-m249): denial of service via globparts amplification |

The rest of the Jest packages (`jest`, `jest-cli`, `@jest/core`, `@jest/reporters`, `jest-config`, `jest-runner`, `jest-runtime`, `jest-circus`) are only flagged because they depend on the packages above.

#### Why it isn't fixed

- npm's only suggested fix is `npm audit fix --force`, which **downgrades `jest` to 29.7.0**. That is a semver-major change and would also require downgrading `jest-environment-jsdom` and `@types/jest`.
- Jest 30 requires `glob@^10`, so the patched `glob`/`minimatch` versions can't be installed without changing Jest itself.

#### Risk assessment

**Low.** These packages:
- only run in the local test runner (`npm test`),
- match the repo's own file patterns and never process untrusted input,
- are not included in the production bundle.

#### Options to resolve

1. **Wait for an upstream Jest 30.x release** that updates `glob` *(recommended)*. Re-run `npm audit` after each Jest update.
2. **Force a patched version with npm `overrides`** in `package.json`:
   ```json
   "overrides": {
     "glob": "<patched version>",
     "minimatch": "<patched version>"
   }
   ```
   This needs verification that Jest still works with the newer major versions (`npm test`).
3. **Downgrade to `jest@29.7.0`** *(not recommended)*. It's a step backwards and would also mean downgrading `jest-environment-jsdom` and `@types/jest`.

When this is revisited, also handle the `ts-jest` mismatch (item D).

### B. `webpack-dev-server` → `sockjs` → `uuid` (3 moderate): resolved

Resolved by upgrading to `webpack-dev-server@6` (decision 5). Original analysis kept below for reference.

```
webpack-dev-server@5.2.7   (latest 5.x, still flagged)
└─ sockjs@0.3.24
   └─ uuid@8.3.2           GHSA-w5hq-g745-h8pq (missing buffer bounds check in v3/v5/v6)
```

**Why it isn't fixed yet:** `5.2.7` is already the newest 5.x release. The only fix is upgrading to `webpack-dev-server@6.1.0`, a semver-major change.

**Risk:** Low. `webpack-dev-server` only runs locally during `npm run dev` and is not part of the production bundle.

**Recommendation:** Upgrade to `webpack-dev-server@6`. Unlike the Jest fix, this is a step forward rather than a downgrade, and `webpack-cli@7.3.1` already supports `webpack-dev-server` `^5.0.0 || ^6.0.0`. Before merging, check the `devServer` block in `webpack.config.js` against the v6 migration guide, then verify `npm test`, `npm run build` and `npm run dev`.

### C. `@babel/core` (1 low): resolved

Resolved by updating `@babel/core` to 7.29.13 (decision 6).

### D. `ts-jest` / `jest` version mismatch: follow-up

`ts-jest@29.4.9` is installed alongside `jest@30.3.0`. Align it to a Jest 30–compatible `ts-jest` when item A is revisited.

## How to check

```bash
cd react-ts
npm audit
npm audit signatures
npm ls jest glob minimatch webpack-dev-server sockjs uuid
```
