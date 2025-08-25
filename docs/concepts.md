# Core Concepts

## Entities
Semantic units (e.g., `ticket`, `product`, `card`, `section`, `button`, `paragraph`).  
`type` is **meaning**, not layout. You may use HTML-like names for simple elements.

**ID scope:** `id` MUST be unique within the **view document**. Use domain IDs inside `props` (e.g., `props.id: "t_8129"`) when stability across views is required.

## Props
Key/values per entity (strings, numbers, booleans, arrays/objects). Keep copy in `props`, optionally with lightweight Markdown (e.g., `title_md`, `content_md`).

## Children
Hierarchy via `children[]` (e.g., section → card → button).

## Affordances (actions)
Explicit operations an agent can perform:
```json
{
  "action": "ticket.update",
  "method": "PATCH",
  "href": "/agent/tickets/{id}",
  "params": { "id": "t_8129", "status": "pending" },
  "dry_run_supported": true
}
```

## Presentation (HTML-only)

```json
"presentation": {
  "tag": "article",
  "variant": "card",
  "tokens": { "tone": "warning", "size": "md" },
  "classes": "rounded border p-4"
}
```

> **Never** expose `presentation` to agents. Strip on the wire.

## Events

Meaningful state changes, not layout:

```json
{"type":"ticket.updated","id":"t_8129","ts":"2025-08-10T18:20:11Z","entity":"ticket","changes":{"status":["open","pending"]}}
```

## Accessibility & HTML mapping

Renderers map semantics to proper HTML/ARIA (`card` → `<article role="region">`, `button` → `<button>`, `section` → `<section>`), complying with WCAG (landmarks, focus, labels).

## Internationalization

Allow `lang` at page/entity and optional dictionaries:

```json
"i18n": {
  "default": "en",
  "strings": {
    "en": { "title": "Buy the ticket" },
    "de": { "title": "Ticket kaufen" }
  }
}
```

## Type Registry & Naming

* **Built-in primitives:** `section`, `paragraph`, `button`, `card`, `image`.
* **Domain types:** free-form (e.g., `ticket`, `product`).
* **Naming:** lowercase kebab or snake case; use dot-namespace for bundles (e.g., `helpdesk.ticket`).
* **Stability:** once introduced on the wire, treat as public contract; deprecate via `ui_hints.deprecated: true` before removal.

> **Images:** Use the `image` node when you need explicit props (src/alt) or future affordances (e.g., open-full). If you only need inline media in copy, a Markdown image inside `*_md` fields is fine.

## Copy Fields & Markdown

* Prefer plain-string fields (e.g., `title`) with optional `*_md` variants for lightweight Markdown. Supported subset: `**bold**`, `*italic*`, `` `code` ``, links `[text](url)`, lists, `##`–`####` headings. Renderers MUST sanitize HTML output. Agents should treat `*_md` as text, not HTML.

## Preconditions Language (normalized)

Affordances may include `preconditions[]`. Each is:

```json
{ "field": "props.status", "op": "equals", "value": "open" }
```

Supported `op`: `equals`, `not_equals`, `in`, `not_in`, `gte`, `lte`, `exists`, `absent`.  
Use `value` for scalar comparisons or `values` (array) for `in/not_in`.

## Audience
Nodes may include an `audience` property with one of `agent | human | both` (default `both`).  
Semantics:
- `human`: node is intended for human rendering only (e.g., navigational header).
- `agent`: node is only for machine consumption (e.g., hidden hints, machine notes).
- `both`: node remains for both audiences.

Inheritance: a parent’s `audience` applies transitively to children unless a child overrides it.
