# Screenshots

These are the images the main [README](../README.md) points at:

| File | Shows |
| :--- | :--- |
| `screenshot-dusk.png` | The **Dusk** theme — also used as the hero image |
| `screenshot-moss.png` | The **Moss** theme |
| `screenshot-parchment.png` | The **Parchment** theme |
| `screenshot-switcher.png` | The document switcher open, with three papers listed |
| `screenshot-shelf.png` | The front page with a populated shelf, showing reading progress |
| `social-preview.png` | The 1280 × 640 card used as the repo's social preview |

The three theme shots show the same page of the same document at the same zoom, so the table in the README reads as a true comparison. The documents in them are synthetic sample papers written for these screenshots — the authors, figures and results in them are invented, not real publications.

## Replacing them

- **Format:** PNG
- **Width:** about 1600px (the current set is 1600 × 914, captured from an 800px-wide window at 2× for retina sharpness)
- **Framing:** include the toolbar, and pick a spread with both body text and a colour figure — the figure is what shows the smart invert doing its job
- **Documents:** have three open, so the toolbar shows the count badge and the switcher has something to list
- **For the shelf shot:** read partway into two of them and reload first, so the progress bars are at different points rather than all empty
- **Consistency:** same document, same page, same zoom and same window size across all three

Keep the filenames as they are and the README picks the new images up.

## The social preview

`social-preview.png` is the card GitHub shows when the repo is linked from Slack, X or LinkedIn. It is set by hand under **Settings → General → Social preview → Upload an image**; there is no API for it, so a fresh clone does not pick it up automatically.

It is built from `social-preview.src.html`, which pulls in `screenshot-dusk.png` alongside it. To change the wording or colours, edit that file, open it in a browser sized to exactly 1280 × 640, and capture the viewport.
