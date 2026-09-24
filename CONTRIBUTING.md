# Contributing

Thanks for taking a look. Lamplight is small and means to stay that way, so contributing is pleasantly low-ceremony.

## Getting set up

```
git clone https://github.com/Vasu021/Lamplight.git
```

Open `index.html` in a browser. That's it — there is no build step and nothing to install.

## Making a change

The whole app is `index.html`. Edit it, reload the browser, and try it against a real PDF.

Before opening a pull request, please walk through the manual checks in [CLAUDE.md](CLAUDE.md#testing): open a PDF by picker and by drag-and-drop, switch between all three themes, zoom in and out, select and copy some text, scroll a long document, and try a file that isn't a PDF.

A few things to keep in mind:

- **No frameworks or build tools.** Being one downloadable file that runs from disk is the point.
- **Nothing leaves the browser.** No uploads, no analytics, no new network requests.
- **Keep it accessible** — visible focus, labels on icon buttons, and the reduced-motion rule.
- **Match the surrounding style**: 2-space indent, plain DOM APIs, calm user-facing wording.

## Opening an issue

Bug reports are very welcome. The useful details are: your browser and OS, what you did, what you expected, and what happened instead. If a specific PDF misbehaves, say what kind it is — scanned, very large, unusual fonts — and attach it if you can share it.

Ideas and "this felt wrong to read" notes are welcome too. The [roadmap](README.md#roadmap) is where the current ideas live; say so if one of them matters to you.

## Adding a theme

Themes are defined in two places that must agree: `PALETTES` in the script and the matching `:root[data-reading="…"]` custom properties in the CSS. See [CLAUDE.md](CLAUDE.md#themes) for the details. A screenshot in the pull request helps a lot.
