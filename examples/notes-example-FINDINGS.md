# notes-example: what "Export notes" does

Source: https://rea.tools/examples/notes-example.zip (unpacked into `notes-example/`).

Traced by reading the source manually. The REA analyzer
(`npx rea-agents analyze-javascript-application`) was not run in this session.

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
