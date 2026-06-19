# vize-vue-ts-sidecar-ts-repro

Minimal reproduction for a bug in the [vize](https://github.com/ubugeeei/vize) VS Code
extension. When the workspace path contains a character that gets percent-encoded in a
`file://` URI (here `=`), opening a `.vue` file makes the LSP write virtual `*.vue.ts`
files next to the source for the whole relative-import closure, and never removes them.

The reproduction lives under `foo=bar-baz/` so the workspace root contains `=`. A path
without such a character does not reproduce — the import surfaces as
`Cannot find module './Child.vue.ts'` (TS2307) instead.

## Prerequisite

Type checking must be reachable: the vize server must resolve a Corsa / `tsgo` runtime
(`@typescript/native-preview`) from the project, a global install, or `TSGO_PATH`. Without
it nothing is written and you get a `typecheck-unavailable` diagnostic.

## Steps

1. Open the `foo=bar-baz/` folder in VS Code with the `ubugeeei.vize` extension (typecheck on).
2. Open `src/Parent.vue`.

## Expected

No files written next to the sources.

## Actual

`src/Child.vue.ts` and `src/Parent.vue.ts` appear on disk (generated "Virtual TypeScript for
Vue SFC Type Checking" files) and are not removed on close / quit / restart.

## Root cause

The server overlays each relatively-imported sibling into its Corsa session at
`<sibling>.vue.ts`.

- `normalize_document_uri` (`corsa_bridge/bridge.rs`) builds the overlay URI by prefixing
  `file://` to the raw path, leaving `=` literal.
- `build_session_document_uri` → `path_to_file_uri` (`file_uri.rs`) percent-encodes every
  byte outside `[A-Za-z0-9-._~/:]`, turning `=` into `%3D`.

So `document_uri != uri`, which skips the `document_uri == external_uri` short-circuit in
`materialize_session_document` (`lsp_client/session.rs`); it then `fs::write`s the virtual
TS to the decoded real path next to the source. The internally-opened siblings are never
closed and the workspace session is not cleaned up on shutdown, so the files persist.

## Environment

vize VS Code extension 0.239.0 (server `vize-maestro 0.239.0`)
