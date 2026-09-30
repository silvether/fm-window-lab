# Window Spawn Lab — Katalon Recorder → FortiMonitor

Single-file app: `index.html`. Children are the same file with `?child=...`. Host it on any HTTPS origin the FortiMonitor probe can reach. `file://` is fine for local recording. It is not fine as the `open` target of a synthetic check.

## Why raw Katalon XML often dies in FortiMonitor

FortiMonitor Browser Synthetic (Multistep) takes Katalon XML under **Check Source → Selenium Source**, but it only executes the published command list. Window-related commands that survive:

- `selectWindow`
- `selectPopUp`
- `deselectPopUp`
- `assertTitle`
- `assertText` / `assertTextPresent`
- `assertAllWindowTitles` / `assertAllWindowNames` / `assertAllWindowIds`
- `pause`
- `waitForPageToLoad` / `waitForText` / `waitForElementPresent`

Commands Katalon loves to emit that FortiMonitor does not list:

- `waitForPopUp`
- `close` / `closeWindow`
- `selectWindow` with `win_ser_1`, `win_ser_2`, … (recorder-local serial ids)
- Studio-only keywords (`switchToWindowTitle`, etc.)

That is the usual translation: record in Katalon, export XML, rewrite the window locator and drop unsupported waits, paste the result.

## Tests on the page

| Id | Control | How it spawns | Window name | Document title | Marker |
|----|---------|---------------|-------------|----------------|--------|
| T01 | `id=t01-open` | `window.open(url, "fmWinReceipt")` | `fmWinReceipt` | `FM Receipt Window` | `MARKER_RECEIPT_OK` |
| T02 | `id=t02-open` | `window.open(url, "_blank")` | none | `FM Blank Window` | `MARKER_BLANK_OK` |
| T03 | `id=t03-open` | `<a target="_blank">` | none | `FM Anchor Blank` | `MARKER_ANCHOR_BLANK_OK` |
| T04 | `id=t04-open` | `<a target="fmWinInvoice">` | `fmWinInvoice` | `FM Invoice Window` | `MARKER_INVOICE_OK` |
| T05 | `id=t05-submit` | `<form target="_blank">` | none | `FM Form Ticket` | `MARKER_FORM_TICKET_OK` |
| T06 | `id=t06-open` | `window.open(url, "fmWinPopup", features)` | `fmWinPopup` | `FM Sized Popup` | `MARKER_POPUP_OK` |
| T07 | `id=t07-open` | `setTimeout(..., 800)` then `window.open` | `fmWinDelayed` | `FM Delayed Window` | `MARKER_DELAYED_OK` |

Parent title is always `FM Window Lab Parent`.

## Recording procedure (FortiMonitor’s own Katalon steps, plus window focus)

1. Allow Katalon to run in private mode. Clear cookies. Open Incognito.
2. Allow pop-ups for the origin that serves this page.
3. Open `index.html`. Start Katalon Recorder. Hit Record.
4. Click **Open named receipt window** (`t01-open`).
5. Click inside the new window. Type `window-focus-confirmed` into `id=child-note`. Click **Acknowledge focus**.
6. Click back on the parent window (or leave it — you will fix focus in the XML).
7. Stop. Export → Format **XML**.

## What to change in the export before FortiMonitor

Replace whatever Katalon used to address the child:

```
selectWindow | win_ser_1
selectWindow | title=FM Receipt Window
```

or

```
selectWindow | name=fmWinReceipt
```

If Katalon inserted `waitForPopUp`, delete it and put this immediately after the click that spawned the window:

```
pause | 800
```

Use `1500` for T07.

Return to the parent with one of:

```
selectWindow | null
deselectPopUp
```

Then `assertTitle | FM Window Lab Parent` so the check fails loudly if focus never came back.

Ready-to-paste copies:

- `fortimonitor-example.xml` — T01 by title, type + assert in the child, `selectWindow | null` home.
- `fortimonitor-popup-example.xml` — T06 by name, `deselectPopUp` home.

Change `YOUR_HOSTED_URL/index.html` in both.

## Locator preference for a probe

1. `title=FM Receipt Window` — stable, visible, survives unnamed `_blank`.
2. `name=fmWinReceipt` — stable when the app assigned a browsing-context name.
3. `selectPopUp` with empty target — first non-top window. Fine for a single popup. Ambiguous if two are open.
4. `win_ser_N` — last resort. Do not ship that to FortiMonitor.

## Serve it

Any static host works. From this directory:

```
python3 -m http.server 8765
```

Then record against `http://127.0.0.1:8765/index.html`. Point the FortiMonitor `open` target at the copy that lives on an origin the probe can see.
