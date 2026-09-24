# Working on Lamplight

Lamplight is a PDF reader that re-renders pages in calm, low-glare colours. Read `README.md` first for what it does; this file is about how to change it.

## Architecture: one file, on purpose

The entire app is `index.html` — markup, CSS and JavaScript in one document, served as a static file. There is no build step, no bundler, no package manager, and no dependency to install.

Keep it that way. The single file *is* the feature: a user can download one `index.html`, open it from disk with no toolchain, and read a PDF. Splitting out `app.js` and `styles.css` would cost that and buy very little at this size. If the file ever grows past the point of comfort, say so and discuss before restructuring.

State lives in three places, and the split matters:

- **In memory** — `docs`, the documents loaded right now, with their pdf.js instances and rendered canvases. Gone on reload.
- **`localStorage`** — theme and zoom. Two small strings.
- **IndexedDB** — the shelf: the PDF bytes and your place in each document, so they survive closing the tab.

## The document model

Lamplight holds several PDFs open at once. Everything hangs off two module-level values:

```js
let docs = [], activeId = null;
```

`docs` is an array of `{ id, name, file, pdf, pages, el, scrollY, laidOut, observer }`. Each document owns its own `<section class="doc">`, its own `IntersectionObserver`, and its own remembered scroll position. Only the active one carries the `.open` class, and only that one is in the layout — the rest are `display: none`, which is why they cost nothing to keep around and why their observers stay quiet.

Each entry of `doc.pages` is `{ page, el, gen }`. `gen` is the render generation its canvas was painted at; `-1` means never painted.

**The order of operations matters when opening.** `baseScale` measures `#shelf`, not the document section, precisely so a hidden document still sizes correctly — but `#shelf` itself is `display: none` until `body.reading` is set. So `body.reading` must go on *before* the first `layoutDoc`, which is why `openFiles` sets it before calling `switchTo`.

## The shelf (IndexedDB)

Documents you open are kept on the device so they come back next visit. Two object stores, deliberately:

| Store | Holds | Written |
| :--- | :--- | :--- |
| `files` | `{ id, blob, fileName }` — the PDF bytes | once, when a document is first opened |
| `meta` | `{ id, name, fileName, size, pages, page, openedAt }` | every time you open one, and as you scroll |

They are separate **so that remembering a page number never rewrites the blob**. Put them in one store and every scroll pause rewrites forty megabytes. The id is `name::size`, the same identity the duplicate check uses.

Rules worth keeping:

- **Every call is non-fatal.** Private browsing, a denied quota, a blocked upgrade — all of it degrades to "the shelf stays empty and reading still works". `remember()` warns once per session, never per file.
- **`rememberPage` reads the page synchronously**, before its `await`. It used to compute `currentPage()` inside the async write, which meant switching documents saved the *new* document's position onto the old one's record.
- **`rememberPage` refuses to write unless `body.reading` is set, and `currentPage()` skips zero-height pages.** A hidden document measures as a stack of empty boxes all at `top: 0`, so every page satisfies the "above the fold" test and the last one wins — which is why reloading from the landing page used to mark the last-read document 100% finished.
- **Nothing is opened automatically on load.** The shelf is a list to choose from; auto-restoring several large PDFs would make the first paint cost seconds for something the reader may not want.
- **Progress is `(page - 1) / (pages - 1)`**, so page 1 reads as 0% rather than `1/pages`.
- **The list is capped at `21.5rem` and scrolls inside itself**, about four and a half rows. The half row is deliberate: it is the cue that there is more. Without the cap a reader with forty documents gets a front page forty rows tall.
- `refreshShelf()` reloads the cached `shelved` array and redraws both the landing list and the switcher menu. Call it after anything that adds, removes or reopens.

## The parts that matter

All the JavaScript lives in one IIFE at the bottom of `index.html`.

- **`recolour(ctx, w, h)`** — the heart of the app. Walks the rendered canvas pixel by pixel and maps it into the current theme. On dark themes it does a *smart invert*: `k = 255 − max(r,g,b) − min(r,g,b)` added to every channel flips lightness while preserving hue, then the result is mapped linearly onto the palette's `bg`→`fg` range so nothing is pure black or pure white. Parchment uses the same linear mapping with `invert: false`. This is the one function where a change is immediately visible on every page — test it against a PDF with colour figures, not just body text.
- **`renderPage(doc, entry)`** — renders one page to a canvas at `baseScale × zoom × devicePixelRatio` (DPR capped at 2.5), recolours it, builds the pdf.js text layer over it, and swaps both into the page element in one go.
- **`layoutDoc(doc)`** — sizes one document's page elements and wires up a fresh `IntersectionObserver` (900px root margin) for it. Called when a document is first shown and whenever it is stale.
- **`invalidate()`** — theme, zoom or width changed, so every page of every document is now stale. Bumps `generation`, clears every `laidOut` flag, and re-lays out only the active document; the others are re-laid out lazily by `switchTo` when you next look at them.
- **`openFiles(list)`** — the entry point for both the picker and drag-and-drop, taking any number of files. Opens each in turn, collects failures rather than aborting on the first, switches to the first success, and reports what went wrong. **`openOne` is where all the user-facing error strings live.**
- **`updatePageLabel(force)` / `goToPage(n)`** — the page counter is an `<input>`, not a label. `updatePageLabel` refuses to overwrite the field while it has focus unless `force` is set, so scrolling never fights what the reader is typing; `force` is used after a jump, where the field must correct itself to the clamped page. `goToPage` clamps to `1…pages.length` and scrolls the page element into view.
- **`goHome()`** — drops `body.reading` so the landing page shows again, *without* unloading anything: it saves the active document's `scrollY` and page first, so returning through the shelf is instant and lands where you left. It hangs off **Your shelf** at the foot of the switcher menu, which is the only way back while reading.
- **`openFromShelf(id)`** — pulls the blob back out of IndexedDB, rebuilds a `File` from it and hands it to `openFiles`, which resumes at the saved page via `doc.pendingPage`.
- **`switchTo(id)` / `closeDoc(id)`** — saving and restoring `scrollY` on the way out and in. `closeDoc` destroys the pdf.js document, disconnects the observer, and falls back to a neighbour; closing the last one returns to the welcome screen.
- **`generation`** — a counter bumped by `invalidate()`. `renderPage` captures it at the start and bails out after each `await` if it no longer matches. This is what cancels stale renders when the user zooms or switches theme mid-render. **Any new `await` inside a render path needs a `if (gen !== generation) return;` after it**, or you will get pages painted in the previous theme. It is deliberately global rather than per-document: a theme change invalidates everything at once.

## Themes

A theme is defined in two places and both must be updated together:

- **`PALETTES`** in the script — `{ bg: [r,g,b], fg: [r,g,b], invert: bool }`, the colours `recolour` maps page pixels onto (page surface and page text).
- **CSS custom properties** under `:root[data-reading="<name>"]` — `--bg`, `--surface`, `--ink`, `--muted`, `--lamp`, `--line`, and optionally `--page-shadow`, the colours for the chrome around the page.

`PALETTES[x].bg` should match `--surface` and `PALETTES[x].fg` should match `--ink`, or the canvas will not sit flush with its page element. A new theme also needs a `.swatch.<name>` gradient rule and a `<button class="swatch …" data-theme="…">` in the toolbar. `dusk` is the fallback when the stored theme is unknown.

## The switcher

The toolbar's filename is a button (`#docsBtn`) that opens a popover (`#docsMenu`) listing every open document. `renderList()` rebuilds the list from `docs` on every change — it is small enough that diffing would be more code than it saves.

Conventions worth keeping:

- The count badge appears only at two or more documents; at one the toolbar looks exactly as it did before the feature existed.
- Document shortcuts are **`Alt`-based**, because `Ctrl`/`⌘` + `Tab` and `+ W` belong to the browser. They are keyed on `e.code` (`Digit1`…`Digit9`), not `e.key`, because `Alt`+`1` does not produce `"1"` on macOS or on many non-US layouts.
- The popover is a `role="menu"` with `menuitemradio` rows: `Escape` closes it and returns focus to the trigger, arrow keys move between rows.
- Opening a file that is already open switches to it instead of loading a second copy, matched on name *and* size.

## The page counter

It is an input styled to look like text until you touch it. Three rules keep it from being annoying:

- **Blur only navigates if the number actually changed.** Otherwise tabbing past the field would snap the reader to the top of the page they were already on.
- **Digits only**, sanitised on `input` rather than rejected on submit, with `maxLength` and a `ch` width derived from the page count.
- **`Escape` reverts** to the real current page rather than committing.

Note for anyone testing this in headless Chrome: `focus()` and `blur()` silently do nothing unless `Page.bringToFront` has been called, so blur-to-commit will look broken when it is not.

## Errors

The welcome screen has an inline error line (`#error`), but once you are reading it is hidden. `notify(msg)` routes to whichever is visible: the inline line when nothing is open, a transient toast (`#toast`) when something is. Opening several files at once collects failures rather than stopping at the first, so one bad file never costs you the good ones.

## pdf.js

Pinned to **3.11.174** from cdnjs, loaded as two `<script>` tags — the library *and* the worker — with `GlobalWorkerOptions.workerSrc` also pointing at the worker URL. Loading the worker as a tag is deliberate: it keeps the reader working when the page is opened over `file://`. If you bump the version, update all three references together and re-test the text layer, which is the part of the pdf.js API most likely to have changed (`renderTextLayer`, `--scale-factor`).

## Conventions

- No frameworks, no build tools, no runtime dependencies beyond pdf.js and the Literata webfont.
- Plain DOM APIs, modern syntax, 2-space indent (see `.editorconfig`).
- Keep it accessible: visible `:focus-visible` outlines, `aria-label` on icon buttons, `aria-pressed` on the theme swatches, `role="alert"` on the error line, and the `prefers-reduced-motion` rule that disables transitions.
- Errors are shown to the user as a calm sentence that says what to do next, never a stack trace or an alert box.
- Nothing may leave the browser. No uploads, no analytics, no telemetry, no new network requests.
- `localStorage` access always goes through the `store` helper, which swallows failures (private browsing).

## Testing

There is no test suite; it is manual, in a real browser. Open `index.html` directly and check:

1. **Open a PDF** — both by picker and by dropping it anywhere on the page.
2. **Open several at once** — select three in the picker, and separately drop three at once. All should appear in the switcher, with the first one shown.
3. **Switch between them** — by clicking a row, by <kbd>⌥</kbd><kbd>1</kbd>…<kbd>9</kbd>, and by <kbd>⌥</kbd><kbd>←</kbd>/<kbd>→</kbd> (which should wrap). The toolbar name, count badge, page counter and window title should all follow.
4. **Scroll memory** — scroll deep into one document, switch away, switch back; you should land where you left.
5. **Close** — close a background document (the one you are reading should not move) and close the active one (you should land on a neighbour). Close them all: you should get the welcome screen back, and be able to open again.
6. **Each theme** — dusk, moss, parchment. Page background, chrome and figures should all change, with no flash of un-recoloured white. Switch theme, then switch to a document you have not looked at since; it should come back in the new theme, not the old one.
7. **The page counter** — on a long document, click it, type a page and press <kbd>Enter</kbd>; you should land on that page. Try a number past the end (clamps to the last page), a `0` (clamps to the first), letters (ignored), <kbd>Esc</kbd> (reverts), and <kbd>↑</kbd>/<kbd>↓</kbd> (steps). Tab into and out of the field without typing — the page must not move. Then scroll by hand and watch the number keep up.
8. **Zoom** — toolbar buttons and <kbd>Ctrl</kbd>/<kbd>⌘</kbd> <kbd>+</kbd>/<kbd>−</kbd>, out to both ends of the 50–300% range. Position should be roughly preserved.
9. **Text selection** — select a paragraph and copy it; the highlight should be theme-coloured and land on the right words.
10. **A long, multi-page PDF** — scroll fast and confirm pages render as they arrive, the page counter keeps up, and switching theme mid-scroll does not leave stale pages behind.
11. **A non-PDF file**, and if you have them a password-protected and a damaged PDF — each should give its own friendly message and leave the app usable. Drop a mix of good and bad files at once: the good ones should still open.
12. **The same file twice** — it should switch to the copy already open, not add a duplicate.
13. **Your shelf (in the switcher)** — read partway in, open the switcher and press **Your shelf**. The landing page should return with that document dotted on the shelf; clicking it should drop you back on the same page with no re-render.
14. **The shelf** — open a document, read a few pages in, then reload. It should be on the shelf with the right page and a part-filled bar, and *not* opened for you. Click it: it should open from storage and land on that page. Reload again after removing it with the × — it should stay gone.
15. **Storage refused** — try it in a private window. The shelf should simply never appear, with one calm message at most, and everything else should work.
16. **Reload** — theme and zoom should come back, and the welcome screen should be clean apart from the shelf.

When testing storage in headless Chrome, remember that a persistent `--user-data-dir` keeps IndexedDB between runs — wipe the profile or the second run starts with yesterday's shelf.

Worth a pass on a narrow window (under 640px the toolbar sheds a separator and the switcher name truncates, but the switcher itself stays — it is the only way to reach other documents) and, if the change touches rendering, on a HiDPI screen.
