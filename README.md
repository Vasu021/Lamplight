# 🪔 Lamplight

**A one-page PDF reader that redraws papers in calm, low-glare colours.**

[![License: MIT](https://img.shields.io/badge/license-MIT-e0b56a?style=flat-square)](LICENSE)
[![No build step](https://img.shields.io/badge/build_step-none-8e939b?style=flat-square)](#how-to-use)
[![Zero dependencies](https://img.shields.io/badge/dependencies-zero-8e939b?style=flat-square)](#how-it-works)

![Lamplight reading a paper in the Dusk theme](docs/screenshot-dusk.png)

| Dusk | Moss | Parchment |
| :---: | :---: | :---: |
| ![Dusk theme](docs/screenshot-dusk.png) | ![Moss theme](docs/screenshot-moss.png) | ![Parchment theme](docs/screenshot-parchment.png) |
| Slate charcoal | Dark green-grey | Warm sepia |

Open as many papers as you like and switch between them from the toolbar:

![The document switcher, listing three open papers](docs/screenshot-switcher.png)

Come back later and your shelf is waiting, with your place in each paper:

![The shelf on the front page, listing three papers with reading progress](docs/screenshot-shelf.png)

## Why

Reading white-background PDFs on a screen — in a viewer, in VS Code, anywhere — is a small floodlight pointed at your face, and after a few papers it shows. Lamplight opens a PDF locally and re-renders every page in a quieter palette, so a long reading session stops being a squinting match. Figures keep their colours and the text stays selectable, so nothing is lost in the trade.

## Features

- **Open anything** — file picker, or drop PDFs anywhere on the page.
- **Many documents at once** — open a stack of papers and switch between them from the toolbar. Each one remembers where you were reading.
- **A shelf that survives closing the tab** — documents you open are kept in this browser, with your place in each. Come back tomorrow, pick one, and carry on from the page you stopped at. Remove any of them in one click.
- **Three reading themes** — Dusk (slate charcoal), Moss (dark green-grey), Parchment (warm sepia).
- **Smart invert** — on dark themes, lightness is flipped but hue is preserved, so a blue bar in a chart stays blue.
- **Text stays text** — selectable and copyable via the pdf.js text layer, with a theme-coloured selection highlight.
- **Fast on long documents** — pages render lazily as you scroll, sharp on HiDPI screens.
- **Zoom 50–300%** — from the toolbar or with <kbd>Ctrl</kbd>/<kbd>⌘</kbd> <kbd>+</kbd> and <kbd>−</kbd>.
- **Jump to any page** — the page counter is editable: click it, type a number, press <kbd>Enter</kbd>. No scrolling through a hundred pages to reach page 94.
- **Remembers you** — theme, zoom and your place in every document persist between visits.
- **Private by design** — the file is read in your browser and never uploaded anywhere.
- **Considerate** — friendly errors, visible keyboard focus, responsive layout, respects `prefers-reduced-motion`.

## How to use

**Online:** <https://lamplight.ink/> — also reachable at <https://vasu021.github.io/Lamplight/>, which redirects there.

**Locally:** download `index.html` and open it in your browser. That's the whole install — there is nothing to build and nothing to `npm install`.

Then choose a PDF or drag one onto the window — drop several at once if you like — and pick a theme from the toolbar at the bottom.

**Your shelf** at the foot of the switcher takes you back to the front page at any time, without closing anything — the documents stay loaded, so stepping back in puts you exactly where you were. A lamp-coloured dot marks the ones still open.

Everything you open goes on **your shelf**, which is waiting on the front page next time you visit — each document with the page you reached and a line showing how far in you are. Click one to carry on where you stopped, or press its **×** to remove it. The list keeps to a fixed height and scrolls once it fills, so forty papers leave the front page as short as three do. Nothing is opened for you automatically; the shelf is a list to choose from, not a session to restore.

To go straight to a page, click the page counter in the middle of the toolbar, type the number and press <kbd>Enter</kbd> — numbers outside the document are clamped to its first or last page, so a mistyped 999 lands on the end rather than doing nothing.

With more than one document open, the filename on the left of the toolbar becomes a switcher: it shows how many are open, and clicking it lists them. Pick one to jump to it, press <kbd>×</kbd> to close it, **Open a PDF…** to add another, or **Your shelf** to step back to the front page. Opening a file you already have open just takes you back to it rather than loading a second copy.

### Keyboard shortcuts

| Keys | Action |
| :--- | :--- |
| <kbd>Ctrl</kbd>/<kbd>⌘</kbd> + <kbd>+</kbd> | Zoom in |
| <kbd>Ctrl</kbd>/<kbd>⌘</kbd> + <kbd>−</kbd> | Zoom out |
| <kbd>Enter</kbd> *(in the page counter)* | Jump to the page you typed |
| <kbd>↑</kbd> <kbd>↓</kbd> *(in the page counter)* | Step forward / back one page |
| <kbd>Esc</kbd> *(in the page counter)* | Cancel and go back to the current page |
| <kbd>Alt</kbd>/<kbd>⌥</kbd> + <kbd>1</kbd>…<kbd>9</kbd> | Jump to the *n*th open document |
| <kbd>Alt</kbd>/<kbd>⌥</kbd> + <kbd>←</kbd> <kbd>→</kbd> | Previous / next document (wraps) |
| <kbd>Esc</kbd> | Close the document switcher |
| <kbd>Tab</kbd> | Move through the toolbar controls |
| <kbd>Space</kbd>, <kbd>↑</kbd> <kbd>↓</kbd>, <kbd>Page Up/Down</kbd> | Scroll the document (ordinary browser scrolling) |

The document shortcuts use <kbd>Alt</kbd>/<kbd>⌥</kbd> rather than <kbd>Ctrl</kbd>/<kbd>⌘</kbd> because <kbd>⌘</kbd><kbd>Tab</kbd> and <kbd>⌘</kbd><kbd>W</kbd> belong to the browser and the operating system.

## How it works

pdf.js renders each page to a canvas, and Lamplight then rewrites the pixels before they reach your eyes.

On the dark themes this is a *smart invert* rather than a plain one. For each pixel it computes

```
k = 255 − max(r, g, b) − min(r, g, b)
```

and adds `k` to all three channels. That flips lightness while leaving the relationship between channels — the hue — intact, which is why charts and figures survive the trip. The result is then mapped linearly onto the theme's background→text colour range, so nothing lands on pure black or pure white; the Parchment theme uses the same mapping without the inversion.

Pages are observed with an `IntersectionObserver` and rendered just before they scroll into view, at `devicePixelRatio` (capped at 2.5) for crispness, and fit to the window width up to 900px.

Every open document keeps its own page list, observer and scroll position; only the active one is in the layout, so the documents you aren't reading cost nothing to keep around. Changing theme or zoom marks all of them stale at once, and each re-renders lazily the next time you look at it.

The shelf uses two IndexedDB stores rather than one: the PDF bytes in `files`, the name, page count and your place in `meta`. They are separate so that remembering a page number never has to rewrite a forty-megabyte blob.

## Privacy

Your PDFs never leave your machine. They are read with the browser's `File` API, decoded in the tab, and drawn to a canvas — there is no server, no upload, no analytics. The only network requests Lamplight makes are for pdf.js and the Literata font, both from public CDNs, the first time you load the page.

The shelf keeps its copies in **IndexedDB**, which is browser storage on your own device, scoped to this one site. Nothing is synced and nothing is sent anywhere — opening Lamplight on another machine gives you an empty shelf. Removing a document deletes Lamplight's copy immediately; your original file on disk is never touched either way. Clearing site data for the page empties the shelf too.

## Limitations

- Scanned (image-only) PDFs have no selectable text — they recolour fine, but there is nothing to select.
- Very large PDFs recolour page by page as you scroll, so expect a brief pause on each new page.
- The first load needs an internet connection to fetch pdf.js and the font.
- Open documents live in memory for the session. A dozen large PDFs at once will use a lot of it, and closing a document is what gives it back.
- The shelf stores a copy of each PDF, so it counts against the browser's storage quota for this site. When there is no room left Lamplight says so and keeps reading — remove something from the shelf to make space.
- The shelf is per browser and per device. It does not follow you to another machine, and clearing site data empties it.

## Roadmap

- Custom theme colours
- Offline / PWA support
- Search within the document, and across the shelf
- Bookmarks and notes within a document

## Contributing

Ideas, bug reports and small patches are all welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

## Deploying your own copy

GitHub Pages will serve this repo as-is:

1. **Settings → Pages**
2. Under **Build and deployment**, set **Source** to *Deploy from a branch*
3. Choose the `main` branch and the `/ (root)` folder, then **Save**

Your copy appears at `https://<username>.github.io/<repo>/` within a minute or two.

This repo is served from a custom domain instead — <https://lamplight.ink/> — which is what the `CNAME` file does. Its Pages address, <https://vasu021.github.io/Lamplight/>, stays live and redirects there. Delete `CNAME` in your fork, or Pages will try to serve your copy at a domain you don't own.

## License

[MIT](LICENSE).

## Acknowledgements

- [pdf.js](https://mozilla.github.io/pdf.js/) by Mozilla (Apache-2.0) does all the hard work of decoding PDFs.
- [Literata](https://fonts.google.com/specimen/Literata) by Google Fonts, a typeface designed for long-form reading.
