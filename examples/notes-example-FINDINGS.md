# notes-example: what "Export notes" does

Source: https://rea.tools/examples/notes-example.zip (unpacked into `notes-example/`).

Analyzed with REA (`rea-agents` 6.5.0):
`npx -y rea-agents@latest analyze-javascript-application <path>/notes-example --json`.
The steps REA did not resolve were checked by reading the source.

## What REA found

`normalized_result.summary.ipc`: 1 literal channel, 1 renderer transmission,
1 main handler, 1 paired transmission, 0 ambiguous. Confidence `derived`, authority `shipped-artifact`.

| REA finding | Location |
|---|---|
| `electron-main` entrypoint `main.js` (via `package.json` `main`) | `package.json:5` |
| `browser-window` that loads `preload.js` and `index.html` | `main.js:15:17-17:4` |
| `context-bridge-api` `notes` with member `exportCsv` | `preload.js:3:0-5:2` |
| `ipc-channel` `notes:export`, renderer side, `invoke`, literal | `preload.js:4:19-4:53` |
| `ipc-handler` `notes:export`, main side, `handle`, inline function | `main.js:8:31-12:1` |
| pairing `notes.exportCsv` -> handler (basis `unique-exact-literal-channel`) | preload.js -> main.js |
| `index.html` loads `renderer.js` | `index.html:11` |

REA could not resolve these parts:
- It saw the `window.notes.exportCsv` read in `renderer.js:2`, but did not link it
  to the bridge API (listed as an `ambiguous-target` unknown).
- It listed the `toCsv(...)` and `writeFileSync(...)` calls inside the handler
  as `dynamic-call` unknowns. It did not resolve the
  `app.getPath("downloads")` value either.

The output path and CSV fields below therefore come from reading the handler
and `csv.js` at the locations REA identified.

## Call path

1. `notes-example/index.html:15` has the button `<button id="export">Export notes</button>`.
2. `notes-example/renderer.js:1-2`: its click handler calls `window.notes.exportCsv()`.
3. `notes-example/preload.js:3-5`: `contextBridge.exposeInMainWorld("notes", ...)` defines
   `exportCsv` as `ipcRenderer.invoke("notes:export")`. This sends an IPC message to the main process.
4. `notes-example/main.js:8-12`: `ipcMain.handle("notes:export", ...)` handles that message:
   - line 9: `destination = path.join(app.getPath("downloads"), "notes.csv")`
   - line 10: `writeFileSync(destination, toCsv(notes), "utf8")`
   - line 11: returns `destination` to the renderer
5. `notes-example/renderer.js:3` shows `Saved to <destination>` in `#status`.

## Where notes.csv is written

It goes to the user's Downloads folder (`app.getPath("downloads")`), for example
`~/Downloads/notes.csv`. The file is UTF-8 and any existing file is overwritten.
There is no save dialog and no error handling: if `writeFileSync` throws, the
invoke promise rejects and the status text is never updated.

## Fields

`notes-example/csv.js:1-8` (`toCsv`):
- header row `id,title` (line 5)
- one row per note: `id` unquoted, `title` always double-quoted with `"` escaped as `""` (lines 2, 6)
- rows joined with `\n` and ending in a trailing newline (line 7)

The data is the in-memory array at `main.js:6`, `[{ id: 1, title: "First note" }]`, so the output is:

```
id,title
1,"First note"
```
