# Better Fountain — repo notes

VS Code extension for the [Fountain](https://fountain.io/) screenwriting markup language. Provides syntax highlighting, autocomplete, outline/structure views, live & PDF preview, statistics, and PDF export. Published on the VS Code Marketplace as `piersdeseilligny.betterfountain`.

Forked from / based on Piotr Jamróz's [afterwriting-labs](https://github.com/ifrost/afterwriting-labs) parser and Matt Daly's [Fountain.js](https://github.com/mattdaly/Fountain.js) preview.

## Layout

```
src/                          extension code (TypeScript, compiled to out/)
├── extension.ts              activate(), document parsing dispatch, view registration
├── commands.ts               command palette handlers (exportpdf, debugtokens, …)
├── afterwriting-parser.ts    fountain → tokens + HTML parser (largest file in repo)
├── token.ts, helpers.ts      token model + operators (is_dialogue, location, …)
├── configloader.ts           reads vscode workspace settings → FountainConfig
├── scenenumbering.ts         standard scene numbering schema (1, 1A, 2, …) + diff-aware updates
├── statistics.ts             dialogue/location/character stats; uses readability-scores
├── utils.ts                  active-doc resolution, scene shifting, openFile (WSL aware)
├── pdf/                      PDF export pipeline
│   ├── liner.ts              token list → page-aware lines
│   ├── pdf.ts                entry: GeneratePdf(); special outputs "$STATS$", "$PREVIEW$"
│   ├── pdfmaker.ts           pdfkit + textbox-for-pdfkit rendering, bookmarks, watermark
│   └── print.ts              a4 / usletter print profiles
├── providers/                vscode language & view providers
│   ├── Completion.ts         autocomplete (characters, scenes, title-page keys)
│   ├── Folding.ts            scene/section folding ranges
│   ├── Symbols.ts            DocumentSymbolProvider (outline)
│   ├── Outline.ts, Characters.ts, Locations.ts, Commands.ts, Cheatsheet.ts
│   │                         tree-data & webview-view providers for the activity-bar panels
│   ├── Preview.ts            live HTML preview webview
│   ├── PdfPreview.ts         pixel-accurate PDF preview webview (uses pdfjs)
│   ├── Statistics.ts         stats webview
│   ├── StaticHtml.ts         "Export HTML" command
│   └── Decorations.ts        editor decorations (dialogue numbers, etc.)
├── courierprime/             bundled default screenplay font (.ttf)
├── noisetexture.png          paper texture for the live preview
├── test/                     jest specs (*.spec.ts) + sample .fountain scripts
└── __mocks__/vscode.ts       stub for unit-testing modules that import 'vscode'

webviews/
├── src/                      preview.html, preview_pdf.html, stats.html/.js, charts/, lib/, pdfjs/
└── stats.webpack.config.js   bundles stats.js (d3 + datatables)

syntaxes/fountain.tmLanguage.json   TextMate grammar
language-configuration.json         brackets, comments, etc.
package.json                        contributes commands, settings, views, keybindings
```

`out/` is the compiled output and is the actual extension entry point (`"main": "./out/extension"`). Both `out/` and `node_modules/` are gitignored.

## Build & test

| Command | Purpose |
|---|---|
| `npm install` | install deps |
| `npm run compile` | full build: `tsc` → webview bundling (webpack) → copyfiles for html/css/fonts/pdfjs/noise into `out/` |
| `npm run watch` | alias for `compile` (no incremental watcher despite the name) |
| `npm test` | run jest specs (`*.spec.ts`, `ts-jest` preset) |
| `npm run open-in-browser` | launch via `@vscode/test-web` for the browser/web extension host |

`F5` in VS Code launches a dev Extension Host (see `.vscode/launch.json`). The "Debug Jest Tests" launch config attaches the Node debugger.

Targets ES2022, CommonJS modules. `tsconfig.json` enforces `noImplicitAny`, `noImplicitReturns`, `noUnusedLocals`, `noUnusedParameters`. Engines: VS Code ≥ 1.74, Node ≥ 18.

## How parsing flows

1. `activate()` registers commands, views, providers, then calls `parseDocument(document)`.
2. `onDidChangeTextDocument` re-runs `parseDocument` on every keystroke for `.fountain` files.
3. `parseDocument` calls `afterparser.parse(text, config, generateHtml)` → `parseoutput { tokens, properties, scriptHtml, titleHtml, lengthAction, lengthDialogue, … }`.
4. The result is cached per-URI in `parsedDocuments` (exported from `extension.ts`). Every other module reads from this map rather than re-parsing.
5. Any open preview webviews receive `updateScript`/`updateTitle` messages with the rendered HTML; the duration status-bar item is refreshed; tree-views (`outline`, `characters`, `locations`) call `.update()`.

PDF export reuses the same parser: `commands.exportPdf` → `afterparser.parse` → `pdf.GeneratePdf` → `liner.line` → `pdfmaker.get_pdf` (pdfkit). `GeneratePdf` is also reused for stats and the in-VSCode PDF preview by passing the sentinel paths `"$STATS$"` or `"$PREVIEW$"`.

## Things that have bitten people / quirks

- **`type` command interception** (`extension.ts:147`). The "parenthetical newline helper" registers a global `type` command. Only one extension can do this at a time — vim/vscodevim conflicts are tracked at <https://github.com/Microsoft/vscode/issues/13441>. The setting `fountain.general.parentheticalNewLineHelper` lets the user disable it.
- **WSL `openFile`** (`utils.ts:350`). Inside WSL, `vscode.env.openExternal` produces a `vscode-remote://` URI Windows can't open. The code shells out to `wslpath -w` + `explorer.exe` instead.
- **Scene numbering uses jsdiff** (`scenenumbering.ts`). When updating numbers it diffs current-vs-required so user-typed custom numbers (anything the schema can't parse) are preserved.
- **`pushSorted` is monkey-patched onto `Array.prototype`** in `afterwriting-parser.ts:14`. Don't rely on it from new modules — keep the patch contained.
- **PDF export filename munging** lives in `commands.exportPdf`: it strips `.fountain`/`.spmd`/`.txt`, appends `(CHAR1,CHAR2,+N)` for character-highlighted exports, then `.pdf`. The `fountain.pdf.defaultExportPath` setting overrides the directory.
- **PDF "highlight changes from a commit"** (`commands.ts` `highlightChanges` branch, added v1.14) depends on the bundled `vscode.git` extension API — declared in `extensionDependencies`.
- **Statistics & PDF preview refresh on save**, not on every keystroke (controlled by `fountain.general.refreshStatisticsOnSave` / `refreshPdfPreviewOnSave`). Live preview *does* update on every keystroke.

## Coding conventions in this repo

- Single-quote strings, 4-space indentation in older files, 2-space in newer ones — match the surrounding file.
- TypeScript style is loose-by-modern-standards: `any` is widespread (especially around the parser's token shape), and several modules use class-with-`any`-fields rather than typed interfaces.
- The parser is largely a port of upstream JS (afterwriting), so code style there is intentionally close to the original — be conservative when editing.
- New settings go in `package.json` `contributes.configuration` AND `configloader.ts` (both the `FountainConfig` class and the `getFountainConfig` mapper). Keep the snake_case ↔ camelCase mapping consistent.
- New commands go in `package.json` `contributes.commands` AND `extension.ts` `activate()` registration. Add a keybinding under `contributes.keybindings` if applicable.

## Per-user reminders

- **Do not commit, push, amend, or open PRs.** The user handles git themselves. Read-only git (`status`, `diff`, `log`, `show`) is fine.
- Prefer type coercion (`Number(x)`, `Boolean(x)`) over `as` casts unless dealing with genuinely-unknown external values.
- Don't run arbitrary `node -e` scripts to probe behaviour — read the code and reason statically.
