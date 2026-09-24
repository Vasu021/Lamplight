# Working on Lamplight

Lamplight is a PDF reader that re-renders pages in calm, low-glare colours. Read `README.md` first for what it does; this file is about how to change it.

## Architecture: one file, on purpose

The entire app is `index.html` — markup, CSS and JavaScript in one document, served as a static file. There is no build step, no bundler, no package manager, and no dependency to install.

Keep it that way. The single file *is* the feature: a user can download one `index.html`, open it from disk with no toolchain, and read a PDF. Splitting out `app.js` and `styles.css` would cost that and buy very little at this size. If the file ever grows past the point of comfort, say so and discuss before restructuring.

## The parts that matter

All the JavaScript lives in one IIFE at the bottom of `index.html`.

- **`recolour(ctx, w, h)`** — the heart of the app. Walks the rendered canvas pixel by pixel and maps it into the current theme. On dark themes it does a *smart invert*: `k = 255 − max(r,g,b) − min(r,g,b)` added to every channel flips lightness while preserving hue, then the result is mapped linearly onto the palette's `bg`→`fg` range so nothing is pure black or pure white. Parchment uses the same linear mapping with `invert: false`. This is the one function where a change is immediately visible on every page — test it against a PDF with colour figures, not just body text.
- **`renderPage(entry)`** — renders one page to a canvas at `baseScale × zoom × devicePixelRatio` (DPR capped at 2.5), recolours it, builds the pdf.js text layer over it, and swaps both into the page element in one go.
- **`layout()`** — sizes every page element, resets the render flags, and wires up a fresh `IntersectionObserver` (900px root margin) so pages render just before they scroll into view. Called on open, on zoom, on theme change, and on debounced resize.
- **`openFile(file)`** — validates the file is a PDF, loads it with pdf.js, destroys any previous document, builds one placeholder element per page, and hands off to `layout()`. All the user-facing error strings live here.
- **`generation`** — a counter incremented by `layout()`. `renderPage` captures it at the start and bails out after each `await` if it no longer matches. This is what cancels stale renders when the user zooms or switches theme mid-render. **Any new `await` inside a render path needs a `if (gen !== generation) return;` after it**, or you will get pages painted in the previous theme.

## Themes

A theme is defined in two places and both must be updated together:

- **`PALETTES`** in the script — `{ bg: [r,g,b], fg: [r,g,b], invert: bool }`, the colours `recolour` maps page pixels onto (page surface and page text).
- **CSS custom properties** under `:root[data-reading="<name>"]` — `--bg`, `--surface`, `--ink`, `--muted`, `--lamp`, `--line`, and optionally `--page-shadow`, the colours for the chrome around the page.

`PALETTES[x].bg` should match `--surface` and `PALETTES[x].fg` should match `--ink`, or the canvas will not sit flush with its page element. A new theme also needs a `.swatch.<name>` gradient rule and a `<button class="swatch …" data-theme="…">` in the toolbar. `dusk` is the fallback when the stored theme is unknown.

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
2. **Each theme** — dusk, moss, parchment. Page background, chrome and figures should all change, with no flash of un-recoloured white.
3. **Zoom** — toolbar buttons and <kbd>Ctrl</kbd>/<kbd>⌘</kbd> <kbd>+</kbd>/<kbd>−</kbd>, out to both ends of the 50–300% range. Position should be roughly preserved.
4. **Text selection** — select a paragraph and copy it; the highlight should be theme-coloured and land on the right words.
5. **A long, multi-page PDF** — scroll fast and confirm pages render as they arrive, the page counter keeps up, and switching theme mid-scroll does not leave stale pages behind.
6. **A non-PDF file**, and if you have them a password-protected and a damaged PDF — each should give its own friendly message and leave the app usable.
7. **Reload** — theme and zoom should come back.

Worth a pass on a narrow window (the toolbar collapses under 640px) and, if the change touches rendering, on a HiDPI screen.
