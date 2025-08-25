# JACE → HTML/ARIA Mapping (Reference)

| JACE type | HTML | Required | Recommended |
|---|---|---|---|
| section | `<section>` | landmark present | aria-labelledby when titled |
| card | `<article>` | role="region" | tabindex="-1" for focus management |
| button | `<button>` | keyboard activation | aria-pressed for toggles |
| paragraph | `<p>/<div>` | semantic grouping | none |
| image | `<img>` | `alt` text required | `loading="lazy"` recommended |
