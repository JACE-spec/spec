# JSON Agent Context Envelope (JACE): Machine-First Pages, Human-Friendly Rendering

> Quick links:  
> • [Getting Started](./getting-started.md) • [Core Concepts](./concepts.md) • [Runtime/Sandbox](./runtime-sandbox.md) • [Security](./security.md) • [Roadmap](./roadmap.md)

## Introduction

Most web UIs are built for people, not agents. Real pages mix content with layout wrappers, utility classes, hydration scaffolding, analytics, and other noise. Agents end up scraping the DOM, guessing parameters, and breaking when a class name changes.

**JSON Agent Context Envelope (JACE)** flips that model. JACE is a **machine-first, JSON representation** of a page’s **entities, state, and affordances (actions)**. From a single source of truth you:

* Return **JACE JSON** to agents for precise, schema-checked interaction.
* Render **accessible HTML** for humans.
* Keep visual details in a **`presentation`** property used only for HTML rendering (never sent to agents).

Think: **semantic JSON for meaning** → **HTML for humans**.

---

## Why machine-first?

* **No guessing:** Actions are explicit (`action`, `method`, `href`, `params`) instead of hidden behind forms or JS handlers.
* **Stable contract:** JSON Schemas, semantic versioning, deterministic pagination, and predictable error models.
* **UI decoupled:** Redesign HTML freely; the JACE contract remains intact.
* **Safer automation:** `dry_run`, audit previews, policy hints, and capability scopes are first-class.

---

## How to use JACE

### Content negotiation

Serve JACE and HTML from the same URL:

```
Accept: application/jace+json  → JACE for agents
Accept: text/html             → HTML for humans
```

* Send `Vary: Accept` so caches handle both.
* Optionally link JACE from your HTML:

```html
<link rel="alternate" type="application/jace+json" href="/page?format=jace">
```

### Core structure (reserved keys)

`version`, `$schema`, `page`, `entities`, `type`, `id`, `props`, `affordances`, `children`, `links`, `summary`, `pagination`, `ui_hints`, **`presentation` (HTML-only)**, and **`audience`** (`agent | human | both`, default `both`).

### Audience-aware responses
Mark nodes that are only useful for humans (e.g., headers, decorative sections) with `audience: "human"`.  
Mark machine-only helper blocks with `audience: "agent"`.  
Renderers and servers should apply audience filtering before delivery, and must never rely on it for access control.

---

## JACE → HTML: practical patterns

### 1) Sections vs. entities

Use `type: "section"` to group child entities without leaking layout semantics.

**JACE**
```json
{
  "$schema": "https://example.com/schemas/jace-view.schema.json",
  "version": "0.1.0",
  "type": "section",
  "id": "overview",
  "children": [
    { "type": "paragraph", "id": "hero_copy", "props": { "title_md": "## Hi there" } },
    {
      "type": "button",
      "id": "cta_btn",
      "props": { "label": "Contact me" },
      "affordances": [
        { "action": "contact.open", "method": "GET", "href": "#contact", "params": {} }
      ]
    }
  ],
  "presentation": { "tag": "section", "classes": "container mx-auto py-12" }
}
```

**Rendered HTML**
```html
<section id="overview" class="container mx-auto py-12">
  <!-- children render here -->
</section>
```

> Rule: **`type` is semantic** (`section`, `ticket`, `product`, …), **not** a visual HTML tag. If you need tag hints, put them in `presentation`.

---

### 2) Buttons = explicit actions

Buttons declare actions in `affordances`. Render as form (no-JS fallback) or JS handler (nicer UX).

**JACE**
```json
{
  "type": "button",
  "id": "buy_ticket_btn",
  "props": { "label": "Buy ticket" },
  "affordances": [
    {
      "action": "ticket.buy",
      "method": "POST",
      "href": "/agent/tickets",
      "params": { "title": "buy ticket" },
      "dry_run_supported": true,
      "requires": ["auth"],
      "scopes": ["tickets:write"]
    }
  ],
  "presentation": { "tag": "button", "classes": "btn btn-primary" }
}
```

**HTML (no-JS)**
```html
<form id="buy_ticket_btn" action="/agent/tickets" method="POST">
  <input type="hidden" name="title" value="buy ticket" />
  <button type="submit" class="btn btn-primary">Buy ticket</button>
</form>
```

**HTML (JS)**
```html
<button id="buy_ticket_btn" class="btn btn-primary">Buy ticket</button>
<script>
  document.getElementById('buy_ticket_btn').addEventListener('click', async () => {
    await fetch('/agent/tickets', {
      method: 'POST',
      headers: {'Content-Type':'application/json','Idempotency-Key': crypto.randomUUID()},
      body: JSON.stringify({ title: 'buy ticket' })
    });
  });
</script>
```

---

### 3) Paragraphs with light Markdown

Store rich copy in `props`; the renderer decides how much Markdown to support.

**JACE**
```json
{
  "type": "paragraph",
  "id": "issue_summary",
  "props": {
    "title_md": "## Checkout failing",
    "content_md": [
      "**Lorem ipsum** dolor sit amet, *consectetur* adipiscing elit...",
      "Lorem ipsum dolor sit amet, consectetur adipiscing elit..."
    ]
  },
  "presentation": { "tag": "div", "classes": "prose max-w-none" }
}
```

**HTML (example)**
```html
<div id="issue_summary" class="prose max-w-none">
  <span name="title"><strong>Checkout failing</strong></span>
  <p name="content_0"><b>Lorem ipsum</b> dolor sit amet, <i>consectetur</i> adipiscing elit...</p>
  <p name="content_1">Lorem ipsum dolor sit amet, consectetur adipiscing elit...</p>
</div>
```

---

### 4) Cards that can act

Bundle data and actions; render as a card for humans; expose explicit affordances for agents.

**JACE**
```json
{
  "type": "card",
  "id": "ticket_card_t_8129",
  "props": {
    "title": "Buy the ticket",
    "description": "Why you might want a new ticket",
    "status": "open",
    "priority": 1,
    "assignee": "me"
  },
  "affordances": [
    {
      "action": "ticket.buy",
      "method": "POST",
      "href": "/agent/tickets",
      "params": { "id": "t_8129", "title": "buy ticket" },
      "dry_run_supported": true,
      "preconditions": [{ "field": "props.status", "op": "equals", "value": "open" }]
    }
  ],
  "presentation": {
    "tag": "article",
    "variant": "card",
    "tokens": { "tone": "neutral", "size": "md", "icon": "ticket" },
    "classes": "rounded-xl border p-4 shadow-sm"
  }
}
```

**HTML**
```html
<article id="ticket_card_t_8129" class="rounded-xl border p-4 shadow-sm">
  <h1 name="title">Buy the ticket</h1>
  <p name="description">Why you might want a new ticket</p>
  <span name="priority">1</span>
  <form id="act_t_8129" action="/agent/tickets" method="POST">
    <input type="hidden" name="id" value="t_8129" />
    <input type="hidden" name="title" value="buy ticket" />
    <button type="submit">buy ticket</button>
  </form>
</article>
```

### 5) Human-only header
```json
{ "type": "section", "id": "header", "audience": "human", "props": { "title": "Site" }, "presentation": { "tag": "header" } }
```

---

## Presentation field (HTML-only)

**Policy:** `presentation` exists **only** to help the HTML renderer choose tags, variants, classes, and design tokens.
**Never expose `presentation` to agents.** It is non-semantic, unstable, and irrelevant to machine behavior.

* **Server rule:** Strip `presentation` (and any `classes`/`style`) from all JACE responses when `Accept: application/jace+json` (or `audience=agent`).
* **Schema rule (wire):** the **agent wire schema forbids `presentation`**.
* **Internal/build schema:** may allow `presentation` for renderers.

**Internal JACE (with presentation)**
```json
{
  "type": "card",
  "id": "ticket_card_t_8129",
  "props": { "title": "Buy the ticket", "priority": 1 },
  "affordances": [
    { "action": "ticket.buy", "method": "POST", "href": "/agent/tickets", "params": { "id": "t_8129" } }
  ],
  "presentation": {
    "tag": "article",
    "variant": "card",
    "tokens": { "tone": "neutral", "size": "md" },
    "classes": "rounded-xl border p-4 shadow-sm"
  }
}
```

**What the agent receives (presentation stripped)**
```json
{
  "type": "card",
  "id": "ticket_card_t_8129",
  "props": { "title": "Buy the ticket", "priority": 1 },
  "affordances": [
    { "action": "ticket.buy", "method": "POST", "href": "/agent/tickets", "params": { "id": "t_8129" } }
  ]
}
```

Implementation hint (pseudo):
```js
function toAgentPayload(jace) {
  return deepOmit(jace, ['presentation', 'classes', 'style']);
}
```

---

## Manifest & discovery

Expose a tiny manifest so agents can discover views, actions, and events.

**Endpoint:** `GET /.well-known/agent-manifest`
```json
{
  "version": "0.1.0",
  "protocols": ["JACE/0.1"],
  "dashboard": { "name": "Your Site", "base_url": "https://example.com" },
  "views": [{ "name": "home", "href": "/agent/views/home" }],
  "actions": [{
    "name": "ticket.buy",
    "method": "POST",
    "href": "/agent/tickets",
    "params_schema_ref": "https://example.com/schemas/ticket-buy.json",
    "returns_schema_ref": "https://example.com/schemas/ticket.schema.json"
  }],
  "events": { "sse": "/agent/events", "types": ["ticket.created","ticket.updated"] },
  "schemas": {
    "jace-view": "https://example.com/schemas/jace-view.schema.json"
  }
}
```

HTML can advertise it:
```html
<link rel="agent-manifest" href="/.well-known/agent-manifest">
<link rel="alternate" type="application/jace+json" href="/?format=jace">
```

---

## Wire-level rules

* **Caching:** `ETag`/`Last-Modified` on views.
* **Pagination:** cursor-based (`pagination.next_cursor`) plus explicit `order_by`/`order_dir`.
* **Idempotency:** support `Idempotency-Key` header for POST/PUT/PATCH.
* **Errors:** standard envelope with codes + hints:

```json
{
  "error": {
    "code": "POLICY_DENIED",
    "message": "Priority 5 requires role: manager",
    "hints": ["Request lower priority", "Use manager credential"]
  }
}
```

---

## Actions: safer by default

Every write action should support `dry_run` and return an `audit_preview`.

**Request**
```http
POST /agent/tickets
Content-Type: application/json
Idempotency-Key: 1b6f1b9d-...
{
  "action": "ticket.buy",
  "params": { "id": "t_8129", "title": "buy ticket" },
  "dry_run": true,
  "intent": "Buy a ticket for the user"
}
```

**Response**
```json
{
  "ok": true,
  "dry_run": true,
  "result": { "id": "t_9001", "status": "open" },
  "audit_preview": "CREATE ticket id=t_9001 title='buy ticket'"
}
```

---

## Events (NDJSON / SSE)

Push only meaningful state changes—no analytics or hydration noise.

```json
{"type":"ticket.updated","id":"t_8129","ts":"2025-08-10T18:20:11Z","entity":"ticket","changes":{"status":["open","pending"]},"recommended_affordances":[{"action":"ticket.assign","method":"PATCH","href":"/agent/tickets/{id}","params":{"id":"t_8129","assignee":"me"}}]}
```

---

## Accessibility & HTML mapping

Your HTML renderer must map JACE semantics to **proper HTML/ARIA** (e.g., `card` → `<article role="region">`, `button` → `<button>`, `section` → `<section>`), with keyboard focus, labels, and landmarks. JACE keeps the meaning; renderers ensure **WCAG-compliant** delivery.

---

## Internationalization

Support `lang` at page/entity level and optional `i18n` dictionaries:
```json
"i18n": {
  "default": "en",
  "strings": {
    "en": { "title": "Buy the ticket" },
    "de": { "title": "Ticket kaufen" }
  }
}
```

---

## Security & stability

* Never allow JS in JACE (including `presentation`).
* Strip `presentation` for agent responses.
* Validate all JACE documents and action payloads with JSON Schema.
* Scope credentials with capabilities; enforce rate limits and approval flows for risky actions.

---

## The future

* **Agent-ready web:** JACE becomes the machine-native layer so agents stop scraping human DOMs.
* **Framework support:** JSX/templating compilers target a JACE IR → HTML for humans, JACE for agents.
* **Agent SEO:** JACE acts like JSON-LD with **actions**—searchable, indexable, and automatable.
* **Safer autonomy:** dry-run, audit, and policy gates are part of the contract, not bolt-ons.

---

## Semantic → HTML/ARIA Mapping (reference)

| JACE type    | HTML             | ARIA/Notes                                    |
| ----------- | ---------------- | --------------------------------------------- |
| `section`   | `<section>`      | consider `aria-labelledby` when titled        |
| `card`      | `<article>`      | `role="region"`, focusable header             |
| `button`    | `<button>`       | keyboard activation, `aria-pressed` if toggle |
| `paragraph` | `<p>` or `<div>` | preserve reading order                        |

Renderers MUST meet WCAG 2.2 AA for focus order, color contrast, and landmark usage.

---

## i18n & Directionality

* `lang` may be set at page/entity. Missing strings fall back in this order: **entity.lang → page.lang → i18n.default → `en`**.
* Direction (`dir`) inferred from `lang` (`ar`, `fa`, `he`, `ur` → RTL) or explicitly overridden in renderer config.
* Translators operate on `i18n.strings[<lang>]` maps; unknown keys are ignored.

---

## Conclusion

JACE cleanly separates **meaning** from **presentation**. Define pages in semantic JSON with explicit affordances; render accessible HTML for people; return clean, schema-validated JACE for agents. The result is a web that humans love to use and agents can reliably operate—no scraping, no guesswork, just intent and action.
