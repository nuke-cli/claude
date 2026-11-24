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

The 10 remaining findings are all the Jest chain (item A). Items F and G are Jest config follow-ups unrelated to the audit. See [Open items](#open-items).

## Ground rules

- **No breaking changes without an explicit decision.** Fixes use `npm update <pkg>` to stay within the version ranges `package.json` already allows. `npm audit fix --force` is not used.
- **Fix by severity, one step at a time.** Critical first, then high, then moderate, each verified separately.
- **Verify every step** with `npm test`, `npm run build` and, when dev-server packages change, `npm run dev`.
- **Changes to `webpack.config.js` are checked in both modes** (`npm run build` and `npm run dev`, including a rebuild), because the config is shared. *(Added after decision 8.)*
- **`engines` must match every locked package.** Check it against each package's `engines.node` after any dependency change. With `engine-strict`, a range wider than the dependencies allow breaks installs. *(Added after decision 8.)*
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

> **Superseded by decision 5.** The `express`, `http-proxy-middleware` and `qs` versions above were correct at this step, but the `webpack-dev-server` 6 upgrade replaced them with new major versions (Express 5, `http-proxy-middleware` 4, `qs` 6.17.1). See decision 5 for what's actually installed. This correction came from the PR #4 review (decision 7).

**Result:** total 17 → 14 (13 moderate, 1 low). `npm test` passes, `npm run build` compiles, and the dev server starts and returns HTTP 200.
**Commit:** `8304f70 fix(deps): patch moderate and low audit vulnerabilities` (together with decisions 5 and 6).

### 5. `webpack-dev-server` 5 → 6: upgraded (2026-11-23)

**Decision:** Accept the semver-major upgrade to `webpack-dev-server@^6.1.0` (resolves open item B). It is a step forward rather than a downgrade, and it is the only fix for the `sockjs` → `uuid` finding.

**What changed:**
- `package.json`: `"webpack-dev-server": "^5.2.3"` → `"^6.1.0"`.
- `sockjs` and `uuid` were removed from the dependency tree. v6 drops SockJS and uses the native `ws` WebSocket server.
- No changes to `webpack.config.js` were needed.

**Transitive major upgrades pulled in by v6.** These replace the versions from decision 4 and were missed in the first version of this log:

| Package | Decision 4 | Installed now |
|---|---|---|
| `express` | 4.22.3 | **5.3.0** (major) |
| `http-proxy-middleware` | 2.0.12 | **4.2.2** (major; requires Node `^22.15.0 \|\| ^24.0.0 \|\| >=26.0.0`) |
| `body-parser` | 1.20.9 | **2.4.0** (major) |
| `path-to-regexp` | (Express 4 version) | **8.4.2** (major; stricter route syntax) |
| `qs` | 6.16.2 | **6.17.1** |

These only run inside the local dev server. The project doesn't define custom Express middleware or proxy rules, so the Express 5 routing changes don't affect it. The verification below was run against this tree.

**Compatibility review** (against the [v6 migration guide](https://github.com/webpack/webpack-dev-server/blob/main/migration-v6.md)):
- The project's `devServer` options (`static`, `hot`, `historyApiFallback`, `port`, `open`) are not affected by any documented v6 breaking change.
- The project doesn't use any of the v6 breaking areas: `proxy` / `bypass`, `setupMiddlewares` / custom Express middleware (v6 moves to Express 5), `server: 'spdy'`, `webSocketServer: 'sockjs'`, or `Server.internalIP`.
- Peer requirements are met: `webpack@5.106.2` (needs `^5.101.0`) and `webpack-cli@7.3.1` (supports `^5.0.0 || ^6.0.0`).
- **New requirement: Node.js `^22.15.0 || ^24.0.0 || >=26.0.0`.** Local development uses Node 24.21.0. `http-proxy-middleware` 4 excludes Node 25, and Jest 30 excludes Node 23. Now enforced in `package.json` (decisions 7 and 8).
- v6 removed the dev server's own CLI flags. The `dev` script (`webpack serve --mode development`) still works because `--mode` is a `webpack-cli` flag.

**Verification:**
- `npm run dev`: compiles; `/` returns 200; an unknown route (`/some/route`) returns 200 with `index.html` (`historyApiFallback` works); the `/ws` endpoint accepts the WebSocket upgrade (HTTP 101) for hot reload.
- `npm test`: 1/1 passing.
- `npm run build`: compiles successfully.

**Result:** total 14 → 11 (10 moderate, 1 low). Only the Jest chain and `@babel/core` remain.
**Commit:** `8304f70 fix(deps): patch moderate and low audit vulnerabilities`.

### 6. Low: fixed (2026-11-23)

**Decision:** Update `@babel/core` within its existing range (resolves open item C).

| Package | Before | After | Issue |
|---|---|---|---|
| `@babel/core` | 7.29.0 | 7.29.13 | Arbitrary file read via a `sourceMappingURL` comment ([GHSA-4x5r-pxfx-6jf8](https://github.com/advisories/GHSA-4x5r-pxfx-6jf8)) |

`7.29.13` was published on 2026-11-04 (19 days before the fix), so it falls outside the very-new-release concern in decision 3.

**Result:** total 11 → 10 (10 moderate, 0 low). `npm test` passes and `npm run build` compiles.
**Commit:** `8304f70 fix(deps): patch moderate and low audit vulnerabilities`.

**Re-check (2026-11-23):** `jest@30.5.2` is the latest release, but its `jest-config` still depends on `glob@^13.0.6`, which is inside the vulnerable range. Upgrading Jest doesn't resolve item A yet.

### 7. PR #4 review fixes (2026-11-23)

A code review of PR #4 found 7 issues. None broke the app at runtime. Five were fixed on `audit-packages`, one commit each:

| # | Finding | Fix | Commit |
|---|---|---|---|
| 6 | *(pre-existing)* `jest.config.json` maps style imports to `identity-obj-proxy`, which wasn't installed, so any test importing `.css`/`.scss` failed | Added `identity-obj-proxy@^3.0.0`. Verified with a temporary test importing `index.scss`: failed before, passes after | `3d78441 fix(test): add missing identity-obj-proxy for style imports` |
| 7 | *(pre-existing)* `devServer.static` pointed at `build/`, which `CleanWebpackPlugin` wipes, so `npm run dev` could serve stale files | Point `static` at `public/`. Verified that `/` serves the in-memory build, routing fallback works, and a stray file in `build/` returns 404 | `415689a fix(dev): serve static files from public instead of build output` |
| 2 | Nothing enforced the Node >= 22.15.0 requirement from decision 5 | Added `"engines": { "node": ">=22.15.0" }`, `.npmrc` with `engine-strict=true`, and `.nvmrc` (`24`). Verified that an unsupported Node version makes `npm install` fail with `EBADENGINE` | `0e46aaa build: require Node.js >= 22.15.0` *(range narrowed in decision 8)* |
| 4, 5 | `BundleAnalyzerPlugin` with `analyzerMode: 'disabled'` produced nothing despite its comment; new lines used spaces instead of tabs | `analyzerMode: 'static'`, `openAnalyzer: false`: each build writes `build/report.html` and exits with no server. Re-indented; `prettier --check` passes | `56e68de fix(build): write static bundle report and fix indentation` *(refined in decision 8)* |
| 1 | The PR description and this log listed pre-upgrade versions for `express`, `http-proxy-middleware` and `qs` | Corrected decisions 4 and 5 and the PR description | `nuke-cli/claude@cd76f51` |

Not fixed: finding 3, tracked as open item E. *(Later shown to be a false positive, see decision 11.)*

The findings and a reply for each were posted on PR #4 as a comment-only review (inline comments with replies linking the fix commits).

### 8. Second PR #4 review fixes (2026-11-23)

A second code review of PR #4 found 4 issues: 1 important and 3 nits. Two came from fixes in decision 7, and one repeated a first-review finding. Three were fixed on `audit-packages`, one commit each:

| # | Finding | Fix | Commit |
|---|---|---|---|
| 1 | *(important; regression from decision 7)* `engines: ">=22.15.0"` was wider than the dependencies allow. With `engine-strict`, Node 23 and 25 passed the project check and then failed with `EBADENGINE` on a dependency: Jest 30 needs `^22.0.0 \|\| >=24.0.0`, and `http-proxy-middleware` 4 needs `^22.15.0 \|\| ^24.0.0 \|\| >=26.0.0` | Narrowed `engines` to `^22.15.0 \|\| ^24.0.0 \|\| >=26.0.0`, the overlap of all dependency ranges. Verified against every locked package: 22.15, 24.0 and 26.0 are accepted with 0 dependency blockers; 22.14, 23.11 and 25.2 are rejected by the project's own check | `5b9b88f build: limit engines to Node versions dependencies support` |
| 2 | *(regression from decision 7)* The shared plugin list meant `BundleAnalyzerPlugin` in static mode regenerated and wrote the report on every dev-server rebuild | `webpack.config.js` now exports `(env, argv) => config` and adds the analyzer only when `argv.mode === 'production'`. Verified: `npm run build` writes the report; `npm run dev` compiled twice (startup plus a rebuild) and wrote no report | `fce6f4b fix(build): run bundle analyzer only for production builds` |
| 3 | *(regression from decision 7)* `npm run build` wrote `report.html` into `build/`, so deploying `build/` as-is would publish the app's module tree | `reportFilename` set to `reports/bundle-report.html` (outside the output folder) and `reports` added to `.gitignore`. Verified: `build/` contains only the app files plus `stats.json` | `2f4b9bc fix(build): write bundle report outside the deployed build folder` |
| 4 | `@types/jest` isn't loaded (`"types": []`). Same as first-review finding 3 | Not fixed. Still tracked as open item E *(false positive, see decision 11)* | n/a |

**Also found by the review:** the `ts-jest` / `jest` mismatch in open item D isn't real. `ts-jest@29.4.9` declares `peerDependencies.jest: ^29.0.0 || ^30.0.0`, and `npm ls ts-jest` reports no peer errors. Item D is closed.

**Accepted for now:** `stats.json` (from `StatsWriterPlugin`) is still written into `build/`. That was the case before this PR and is out of scope here. *(Reversed in decision 10: moved to `reports/`.)*

**PR #4 updated:** the description reflects the new Node range and the analyzer behaviour, and drops the `ts-jest` note. The 4 findings and their replies were posted as a second comment-only review.

**Lesson:** two of the three issues were introduced by the previous round of review fixes. Fixes to build tooling and `engines` should be checked against both `npm run build` and `npm run dev`, and an `engines` range should be checked against every locked package.

### 9. No source maps in production builds (2026-11-24)

**Decision:** Set `devtool: false` for production builds. Development keeps `'source-map'`.

**Why:** `devtool: 'source-map'` applied to every mode, so `npm run build` wrote `build/main.<hash>.js.map` with the full original `src/**` code in `sourcesContent`. Deploying `build/` published the source code. Found by the third PR #4 review (it was there before this PR).

**Options considered:**

| Option | Result | Chosen |
|---|---|---|
| `devtool: false` for production | No `.map` files and no `sourceMappingURL` in the bundle; source isn't published | ✅ |
| `'hidden-source-map'` | No reference in the bundle, but the `.map` files (with full source) are still written to `build/` and deployed | ❌ |
| Keep `'source-map'` | Full source published with the app | ❌ |

**Trade-off accepted:** errors from the deployed app show minified stack traces (line 1, column N) that can't be mapped back to the source.

**If readable production stack traces are needed later:** generate source maps outside `build/` (or delete them after upload) and upload them to an error tracker. Don't deploy them.

**Verification:** after `npm run build`, `build/` has no `.map` files and the bundle has no source-map reference. The only `sourceMappingURL` text in it is a string inside `style-loader`'s runtime, used only for CSS source maps. `npm run dev` still serves `main.<hash>.js.map` (HTTP 200).

**Commit:** `cee0d2b fix(build): disable source maps for production builds`. The current form is in `f551407` (`devtool: isProduction ? false : 'source-map'`).

**Documented in:** the PR #4 description (new "Behaviour change" section).

### 10. Third PR #4 review fixes (2026-11-24)

A third code review of PR #4 found 7 issues: 1 important, 3 pre-existing and 3 nits. Two of the nits came from the analyzer change in decision 8. Four were fixed on `audit-packages`, in 3 commits:

| # | Finding | Fix | Commit |
|---|---|---|---|
| 1 | *(important)* `StatsWriterPlugin` (assets, chunks, modules) still wrote `build/stats.json`, so deploying `build/` published dependency paths such as `node_modules/react/index.js` after the bundle report had been moved out | Write it to `reports/stats.json` (git-ignored), next to the bundle report. Verified: same content, `build/` no longer has it, the dev server returns 404 for it | `e500391 fix(build): write stats.json outside the deployed build folder` |
| 2 | *(pre-existing)* `devtool: 'source-map'` in every mode published the full source in `build/*.js.map` | `devtool: false` for production. See decision 9 | `cee0d2b fix(build): disable source maps for production builds` |
| 5 | *(regression from decision 8)* The exported function pushed onto a shared `config.plugins`, so calling it twice for production stacked two analyzers | Return a new config object on every call. *(Completed in decision 11: plugin instances were still shared)* | `d718fee refactor(build): derive mode once and stop mutating shared config` |
| 6 | *(regression from decision 8)* `mode: 'development'` was hardcoded while the analyzer checked `argv.mode`, so `webpack serve --mode production` regenerated the report on every rebuild | Read the mode once (`argv.mode`, default `'development'`) and use it for `mode`, `devtool` and the analyzer. Skip the analyzer under `webpack serve` (`env.WEBPACK_SERVE`, confirmed set by webpack-cli 7). Verified: `webpack serve --mode production` compiled and rebuilt without writing a report | `d718fee` (same commit) |
| 3 | *(pre-existing)* `collectCoverageFrom` only covers `src/{components,containers}/**`, and neither folder exists, so coverage always reports 0 files | Not fixed. Jest config, unrelated to the audit. Open item F | n/a |
| 4 | *(pre-existing)* The `@components` entry in `moduleNameMapper` isn't anchored: any import containing `@components` is redirected, and `@components/Button` resolves to the folder | Not fixed. Jest config, unrelated to the audit. Open item G | n/a |
| 7 | `@types/jest` isn't loaded. Third time reported | Not fixed at the time (item E). Shown to be a false positive in decision 11 | n/a |

**Verification:** `npm run build` writes `bundle-report.html` and `stats.json` to `reports/`, and `build/` holds only `index.html`, the bundle and its license file. `npm run dev` loads, routing fallback works, source maps are served and no report is written. Tests pass, Prettier passes.

### 11. Fourth PR #4 review fixes (2026-11-24)

A fourth code review of PR #4 found 5 issues: 2 pre-existing and 3 nits, none blocking. This reviewer ran the build, the tests and the dev server. All five were addressed:

| # | Finding | Fix | Commit |
|---|---|---|---|
| 1 | *(pre-existing)* `CleanWebpackPlugin` also ran under the dev server and emptied `build/` on disk, so every `npm run dev` deleted the last production build | Add `CleanWebpackPlugin` only when not running under `webpack serve`. Verified: `build/` survives `npm run dev` | `f551407 fix(build): build plugins per mode and keep dev server off build/` |
| 4 | *(from decision 10)* The "fresh config" comment overstated it: each call returned a new object, but plugin instances were created once at module load and shared | Create the whole config, including plugin instances, inside the exported function. Verified: two calls share no objects | `f551407` (same commit) |
| 5 | `StatsWriterPlugin` also ran under the dev server, writing `stats.json` into memory on every rebuild where nothing reads it | Add it only for production builds (`isProduction && !isDevServer`), like the analyzer | `f551407` (same commit) |
| 2 | *(pre-existing)* `tsc --noEmit -p .` failed with TS2882 on `import './index.scss'` (`noUncheckedSideEffectImports` with no stylesheet declaration) | Added `src/global.d.ts` declaring `*.scss` and `*.css`. Verified: `tsc` exits 0, and an import of a missing non-stylesheet module still fails with TS2882 | `2922165 fix(types): declare stylesheet modules for tsc` |
| 3 | The PR description didn't mention that production builds no longer have source maps | Added a "Behaviour change" section to the PR #4 description and recorded decision 9 | `nuke-cli/claude@9d70b70` |

**Production output unchanged:** after `f551407` the production bundle hash is the same as before the change (`main.bd5a611…`).

**Item E closed as a false positive.** Checked while reviewing: `@types/jest` *is* loaded. `@testing-library/jest-dom`'s type files reference it, so `"types": []` doesn't keep it out. `tsc --listFiles` includes `node_modules/@types/jest/index.d.ts`, and `tsc` reports no `test`/`expect` errors in `src/App.test.tsx`. Every earlier review (and the replies saying "tracked as item E") had this wrong.

**Lesson:** a claim repeated across reviews isn't evidence. Check it (here with `tsc --listFiles`) before tracking it as an open item.

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

If Jest is upgraded or downgraded when this is revisited, re-check the `engines` range (decision 8): Jest 30's Node range is one of the inputs to it.

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

### D. `ts-jest` / `jest` version mismatch: closed (not an issue)

`ts-jest@29.4.9` is installed alongside `jest@30.3.0`, which looked like a mismatch. It isn't: `ts-jest@29.4.9` declares `peerDependencies.jest: ^29.0.0 || ^30.0.0`, and `npm ls ts-jest` reports no peer errors. Found by the second PR #4 review (decision 8).

### E. `@types/jest` isn't loaded by TypeScript: closed (false positive)

Reported by the first three PR #4 reviews: `tsconfig.json` sets `"types": []`, which turns off automatic loading of `@types/*` packages, so `@types/jest` was assumed to have no effect.

It does load: `@testing-library/jest-dom`'s type files reference `@types/jest`, so it's included regardless of `"types"`. `tsc --listFiles` lists `node_modules/@types/jest/index.d.ts`, and `tsc --noEmit -p .` passes with no errors in `src/App.test.tsx`. Closed in decision 11. No change needed.

### F. Coverage reports 0 files: follow-up

*(Pre-existing; third PR #4 review, finding 3.)* `jest.config.json` sets `collectCoverageFrom` to `src/{components,containers}/**`, and neither folder exists, so coverage is always empty.

**Fix:** widen it to `src/**/*.{ts,tsx}` and exclude test files (`!src/**/*.test.{ts,tsx}`) and declaration files (`!src/**/*.d.ts`). Out of scope for the audit PR.

### G. Unanchored `@components` mapper: follow-up

*(Pre-existing; third PR #4 review, finding 4.)* The `@components` key in `moduleNameMapper` has no anchors, so any import path containing `@components` is redirected, and `@components/Button` resolves to the folder instead of the file.

**Fix:** replace it with `^@components$` and `^@components/(.*)$` mapping to `<rootDir>/src/components/$1`. Out of scope for the audit PR.

## How to check

```bash
cd react-ts
npm audit
npm audit signatures
npm ls jest glob minimatch webpack-dev-server sockjs uuid
```
