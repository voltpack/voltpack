# Voltpack

- **Status:** planned — do not implement until asked
- **Package:** `@voltpack` (bundler) / `create-voltpack` (scaffold)
- **Date:** 2026-09-09

Voltpack is a bundler built on **Bun’s bundler**. It keeps Bun’s speed and does not replace Bun. It only owns the project-facing layer: one config file, one resolve graph, one plugin surface.

## Goal

Bundling with Bun today often needs extra files (`bunfig.toml`, a wrapper plugin module, ad-hoc `onResolve` hacks). Voltpack should make that unnecessary.

The app has **one** config:

- `voltpack.config.ts`
- or `voltpack.config.js`

No `bunfig.toml`. No extra plugin entry file. Framework and resolve rules live in that config.

## Problems Voltpack must own

### Duplicate dependencies (`file:` / `link:` / peer packages)

This is the same break as Sinwan + `sinwan-router` (or any package that depends on `sinwan`).

Bun and Vite resolve from the **importer**. A linked package with its own `node_modules/sinwan` loads a **second runtime**. Signals, `inject`, and component identity then fail.

Today’s workaround:

- `bun-plugin-sinwan` pins `sinwan` / `sinwan/*` from the app root (`onResolve`)
- `vite-plugin-sinwan` uses `resolve.dedupe: ["sinwan"]`

That is a plugin-level patch. **Voltpack is the place that should fix this for every package**, not only Sinwan:

- Resolve singleton packages from the **app root**, not from a linked package’s `node_modules`
- Deduplicate peers and `file:` / `link:` graphs the way Vite `resolve.dedupe` does, without a per-framework wrap
- Keep one copy of the framework runtime in the bundle (Sinwan, and later any similar library)

See `sinwan-agent/bun-plugin-sinwan/dedupe-linked-sinwan-runtime.md` for the current plugin fix. Do not add more app-level `setup` wrappers.

### Other problems the same layer should absorb

- Extra config files (`bunfig.toml`, `sinwan-plugin.ts` only to pass options)
- Inconsistent resolve between `bun run dev` and `bun build`
- Linked workspace packages resolving the wrong `node_modules`
- Framework plugins (compiler, HMR, JSX) registered in one place instead of scattered Bun plugin objects
- Dev vs production using different entry points and silently different graphs

## Config shape (intent, not API)

`voltpack.config.ts` / `voltpack.config.js` is the only required project file besides `package.json`.

Intended contents (names can change when the package is built):

- entry / outdir
- framework plugins (e.g. Sinwan)
- `dedupe` / `resolve` (app-root pinning for singleton packages)
- alias, target, minify, define
- server / HMR options if Voltpack owns `dev`

Until Voltpack exists, Sinwan apps keep using `bun-plugin-sinwan` / `vite-plugin-sinwan`.

## Out of scope until implementation starts

- Do not write the bundler, CLI, or config loader yet
- Do not replace `bun-plugin-sinwan` or `vite-plugin-sinwan` yet
- Do not add Voltpack to `create-sinwan` templates yet

When work starts, implement in `voltpack/` and record a real fix note. Update this file instead of creating a second vision doc.
