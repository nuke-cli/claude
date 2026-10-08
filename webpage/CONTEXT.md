# nuke-cli webpage: working context

Exported from a Claude Code session on 2026-10-08. It records the decisions and the current state of the `nuke-cli/webpage` project so later sessions can pick it up without the original conversation.

## Project

- Landing page for [nuke-cli](https://www.npmjs.com/package/nuke-cli) (npm package, repo `cl4pper/nuke-cli`).
- Stack: Svelte 3, TypeScript, SCSS, Rollup 2 (IIFE bundle in `public/build/`), served with `sirv-cli`.
- Runs in Docker: `npm run app:up` builds the `svelte-application` container on **port 4000** (`LOCAL_PORT` / `DOCKER_PORT` in `.env`).
- Node is **not installed on the dev machine**. Run npm commands in a container:
  `docker run --rm -v "$PWD":/app -v nuke_nm:/app/node_modules -w /app node:24-alpine npm run build`
- The webpage repo has a `CLAUDE.md` with commands and architecture notes.

## Design source

- `webpage/nuke-cli_home.jpg` is the home page wireframe (1280×832, greyscale).
- **Every grey circle in the wireframe is a placeholder.** That covers the npm/GitHub icons and the logo next to the title. No nuke-cli logo exists yet, so the circles stay grey until assets are added here.

## Decisions

| Topic | Decision |
|---|---|
| Node version | Node 24 LTS: `node:24-alpine` in the Dockerfile, `"engines": { "node": ">=24" }` in package.json |
| Footer stats | **Hardcoded**: version `1.1.8`, downloads `205`. Not fetched from npm; update them by hand. |
| Icons/logo | Keep the wireframe placeholders. No real npm/GitHub logos. |
| Tagline | "A simple way to start **your** next web project." (the wireframe's "you" was a typo) |
| GitHub link | `https://github.com/cl4pper/nuke-cli` (the repository URL in the package's npm metadata) |
| Font | Inter (Google Fonts), weights 400/700 |

## Implemented (home page)

- Components under `src/components/<Name>/component.svelte`, re-exported from `src/components/index.ts`:
  - `NavLink`: grey pill link with a placeholder circle; opens in a new tab.
  - `InstallCommand`: `$ npm install -g nuke-cli` box with a copy-to-clipboard button that shows a checkmark for 1.5s.
  - `Stat`: value and label pair for the footer.
  - `Text`: pre-existing, sizes sm/md/lg.
- `src/App.svelte` lays out the nav (top right), the centred hero and the grey footer band. Phone styles apply at ≤480px and were checked at 375px and 320px.
- `public/global.css` holds the colour variables (`--color-surface: #d9d9d9`, etc.) and a CSS reset.

## Fixes made along the way

- Rollup didn't resolve the `@utils` / `@components` aliases. Added `@rollup/plugin-alias`, and these must stay in sync with `tsconfig.json` `paths`.
- `classnames()` added literal `false` classes. It now keeps only the classes whose condition is true.
- The Dockerfile never ran the build, so the container served a blank page. Added `RUN npm run build`. Source changes need `docker compose up -d --build`.
- `global.d.ts` was added to `tsconfig.json` `include`, which removes the TS2307 `.svelte` module warnings.

## Status / open items

- Home page changes are **not committed yet** in the webpage repo.
- The copy button and the nav links haven't been clicked in a real browser. They were checked only with headless screenshots.
- Known leftover warnings: Dart Sass legacy JS API deprecation, and Node's `url.parse()` deprecation from `sirv-cli@1`. Upgrading sirv-cli would remove the second one.
- Waiting on a nuke-cli logo and any further page designs. Add them to this folder.
