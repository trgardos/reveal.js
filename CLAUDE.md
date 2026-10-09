# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Requires Node >= 22.12.

- `npm start` — Vite dev server on port 8000 (`npm start -- --port=8001` to change). Serves `index.html`, `demo.html`, `examples/` and `test/` straight from source.
- `npm run build` — `tsc` type-check, then the core bundle, the styles and each plugin (each plugin has its own `plugin/<name>/vite.config.ts`), all written to `dist/`.
- `npm run build:styles` — rebuild only `css/` into `dist/`.
- `npm run build:es5` — full build plus a legacy ES5 UMD bundle (`dist/reveal.es5.js`).
- `npm test` — runs every `test/*.html` QUnit page in headless Chrome (Puppeteer) against a temporary Vite server on port 8009. CI runs `npm run build` before `npm test`.
- To run one test, open it in a browser: `npm start`, then visit `http://localhost:8000/test/test-state.html`. `scripts/test.js` has no filter flag.
- React wrapper: `npm run react:build`, `npm run react:test`, `npm run react:demo`.

There is no lint script. Formatting follows `.prettierrc`: tabs, single quotes, 100 columns. CONTRIBUTING asks that changes follow the style of the file being edited (tabs and single quotes in particular).

## Architecture

**The core is a factory, not a class.** In `js/reveal.js`, `export default function( revealElement, options )` builds a closure that holds all deck state (indices, current slide, `dom`, `config`) and returns the `API` object defined near the end of the file. Every presentation is a separate closure, so several decks can run on one page (`test/test-multiple-instances.html`).

**`js/index.ts` is the public entry point.** It wraps the factory in a singleton shim for the old `Reveal.initialize()` API from before 4.0. Calls such as `on` or `registerPlugin` made before `initialize` are queued and replayed once it runs. `new Reveal(el, opts)` is the multi-instance path.

**Controllers.** `js/controllers/*.js` each handle one concern (backgrounds, fragments, keyboard, location/hash, scroll view, print view, auto-animate and others). `reveal.js` creates each one with `new X( Reveal )`, passing the instance's API. Controllers read state through that API and do not import each other. `reveal.js` drives them through methods like `update()`, `sync()` and `configure()` when the deck changes. `js/components/playback.js` is the auto-slide progress UI.

**Mixed JS and TypeScript.** The runtime is mostly plain JS. TypeScript covers the config (`js/config.ts` holds `RevealConfig` and `defaultConfig`), the utilities (`js/utils/*.ts`), the hand-maintained API types (`js/reveal.d.ts`), and each plugin's `index.ts`. A plugin's `index.ts` adds types around its `plugin.js` implementation. When you add or change a config option or an API method, update `config.ts` / `reveal.d.ts` as well.

**Plugins** (`plugin/{highlight,markdown,math,notes,search,zoom}`) are factories that return `{ id, init(deck) }` and are loaded by `js/controllers/plugins.js`. Each one builds to its own ES and UMD bundle and has a subpath export in `package.json`. Do not add new third-party plugins to this repo; CONTRIBUTING says they belong in separate repositories.

**Styles.** `css/reveal.scss` is the core stylesheet. `css/theme/*.scss` are themes compiled by `vite.config.styles.ts` (see `css/theme/README.md`).

**Tests** are standalone HTML pages. Each one imports `reveal.js` and `reveal.css`, which Vite aliases (in `vite.config.ts`) map to `js/` and `css/`. Tests therefore run against source rather than `dist/`, except `test-mathjax4.html`. To add tests, create a new `test/test-*.html` page or add `QUnit.test` blocks to an existing one. The test runner picks up new pages automatically.

**`dist/` is committed.** Rebuild it only when a change is meant to ship built output.

**React wrapper** (`react/`) is the separate `@revealjs/react` package, with its own `package.json` and Vitest tests. Before changing it, read `react/AGENTS.md`. It covers lifecycle and sync rules: one Reveal instance per `Deck`, StrictMode safety, call `sync()` only when the slide structure changes, and shallow-compare `config` before calling `configure()`.
