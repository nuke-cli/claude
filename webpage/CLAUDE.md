# webpage

Context for the **nuke-cli webpage** repository (`github.com/nuke-cli/webpage`, local checkout at `../webpage`).

The webpage is the landing page for [nuke-cli](https://www.npmjs.com/package/nuke-cli), the CLI that scaffolds new web projects (`npm install -g nuke-cli`, then `nuke start:ts <name>`). The CLI lives in `../cli`; the React + TypeScript template it generates lives in `../react-ts`.

## Goal

A single landing page that tells visitors what nuke-cli is, how to install it, and where to find it.

## Wireframe

Desktop: [`nuke-cli_home.jpg`](./nuke-cli_home.jpg). There is no mobile wireframe yet.

From top to bottom:

1. **Header (top right):** two pill buttons with icons, **npm** and **GitHub**, linking to the npm package and the repository.
2. **Hero (centered):**
   - Title `nuke-cli` in large bold type, with a round logo placeholder to its right.
   - Tagline: "A simple way to start your next web project." (the wireframe has a typo, "you next").
   - Install command box: a muted `$` prompt, then `npm install -g` followed by **`nuke-cli`** in bold, and a copy-to-clipboard icon on the right. The box has a dark border and rounded corners.
3. **Footer stats bar (full width, light grey):** two stats, each a large bold number above a small label:
   - `1.1.8` / "latest version"
   - `205` / "total downloads"

Style: monochrome greys with black text, a sans-serif font (Inter-like), generous whitespace and rounded corners.

Open questions:
- Should the version and download count be fetched live (npm registry and npm downloads API), or hardcoded? Live data is the intent implied by the design.
- What should replace the logo placeholder?
- Mobile layout.

## Tech stack

| Concern    | Choice                                                                |
| ---------- | --------------------------------------------------------------------- |
| Framework  | Svelte 3 (`<script lang="ts">`, `<style lang="scss">`)                |
| Language   | TypeScript 5, extending `@tsconfig/svelte`                            |
| Bundler    | Rollup 2 → `public/build/bundle.{js,css}` (IIFE)                      |
| Styling    | SCSS via `svelte-preprocess` + `sass`, component-scoped               |
| Serving    | `sirv-cli` on port 4000                                               |
| Lint       | Prettier (+ `prettier-plugin-svelte`), run by a Husky pre-commit hook |
| Container  | Docker (`node:17-alpine`) + docker-compose                            |

## Scripts

| Script       | What it does                                                       |
| ------------ | ------------------------------------------------------------------ |
| `app:dev`    | `rollup -c -w`: watch build, serves on :4000 with livereload        |
| `build`      | Production build (minified with terser)                            |
| `start`      | `sirv public --host 0.0.0.0 --port 4000`                           |
| `app:up`     | `docker-compose up -d`                                             |
| `app:down`   | `docker-compose down`                                              |
| `lint`       | `prettier --write` on `.svelte` / `.ts` files                      |

## Project structure

```
src/
  main.ts                  # mounts App on document.body
  App.svelte               # root component (still the "Hello, world" placeholder)
  components/
    index.ts               # barrel: export { default as X } from './X/component.svelte'
    Text/component.svelte  # <Text size="sm|md|lg" bold italic> wrapping <p>
  utils/
    index.ts               # barrel
    classnames.ts          # classnames(main, [{ class, condition }])
public/
  index.html               # loads global.css, build/bundle.{js,css}
```

## Conventions

- **Path aliases:** `@components` and `@utils` (set in `tsconfig.json`). Import from the barrels, not from deep paths.
- **Components:** one folder per component, `ComponentName/component.svelte`, re-exported from `components/index.ts`.
- **CSS classes:** BEM-ish, `.Block` and `.Block--modifier` (for example `.Text--bold`), built with `classnames`.
- **Prettier:** tabs, width 2, print width 100, single quotes, semicolons.
- **Git:** [Conventional Commits](https://www.conventionalcommits.org/). Work on `develop` and merge into `main` through PRs. A remote `webpage-02` branch also exists.

## Current state

The repo is scaffolding only, last touched in Nov 2024. None of the wireframe is built yet: `App.svelte` still renders "Hello, world" and `Text` is the only component.

## Known issues and gotchas

- `classnames` produces stray `false` tokens in the class string: `curr.condition && curr.class` evaluates to `false` when the condition fails. Use `curr.condition ? curr.class : ''`.
- `Text` marks `size` as required but also falls back to `md` when it is missing. It should probably be optional with a default of `'md'`.
- The `Dockerfile` has `EXPOSE 8080`, but `start` serves on 4000. The ports actually used come from `.env` (`DOCKER_PORT`, `LOCAL_PORT`).
- `package.json` `repository`, `bugs` and `homepage` still point to the `cl4pper/svelte-setup` template.
- `node:17-alpine` is end-of-life, and Svelte 3 and Rollup 2 are both several major versions behind.
- `.gitignore` excludes lockfiles (`*lock*`), so installs are not reproducible.
