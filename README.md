# Why API and types matter

Slidev presentation for the LMS Seminar, 18 March 2026.

## Content

- `slides.md` — main deck (title, outline, section transitions)
- `pages/api.md` — API design slides
- `pages/types.md` — type system slides
- `pages/footer.md` — summary, conclusions, references

## Run locally (with sources)

```bash
bunx slidev
```

Opens at http://localhost:3030. Supports hot-reload — edits to the `.md` files update the slides live.

## Pre-built exports

Pre-built exports are in `exports/`:

- **`slides-spa.zip`** — interactive SPA with animations. Unzip and serve:
  ```bash
  cd exports
  unzip slides-spa.zip
  python -m http.server --directory dist
  ```
  Then open http://localhost:8000.

- **`slides-export.pdf`** — PDF (one page per slide)
- **`slides-export.pptx`** — PowerPoint (slides as images)

## Rebuild exports

Requires `bun` (or `npm`/`pnpm`).

```bash
bunx slidev build                                  # SPA → dist/
bunx slidev export                                  # PDF
bunx slidev export --format pptx                    # PPTX
```

## License

MIT
