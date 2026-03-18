# Why API and types matter

Slidev presentation for the LMS Seminar, 18 March 2026.

## Content

- `slides.md` — main deck (title, outline, section transitions)
- `pages/api.md` — API design slides
- `pages/types.md` — type system slides
- `pages/footer.md` — summary, conclusions, references

## View the presentation

Run the dev server:

```bash
bunx slidev
```

Opens at http://localhost:3030.

## Exports

Pre-built exports are in `exports/`:

- `slides-spa.zip` — interactive SPA (unzip, then serve with `python -m http.server --directory dist` and open http://localhost:8000)
- `slides-export.pdf` — PDF with click steps
- `slides-export.pptx` — PowerPoint (slides as images)
