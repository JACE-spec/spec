# Getting Started

## Content negotiation

```
Accept: application/jace+json                  → JACE for agents
Accept: application/jace+json;profile="…/0.1" → JACE with explicit profile
Accept: text/html                             → HTML for humans
Vary: Accept
```

Optional: `?format=jace` and:

```html
<link rel="alternate" type="application/jace+json" href="/page?format=jace" />
```

## Core structure (reserved keys)

`version`, `$schema`, `page`, `entities`, `type`, `id`, `props`, `affordances`, `children`, `links`, `summary`, `pagination`, `ui_hints`,  **`presentation` (HTML-only; never for agents)**  and **`audience`** (`agent | human | both`, default `both`).

## Reserved Keys — Formal Definitions

* `links[]`: array of link objects `{ rel, href, title?, method?, type? }`. `rel` SHOULD be from the [IANA Link Relations](https://www.iana.org/assignments/link-relations/link-relations.xhtml) registry; vendor-specific relations MAY use `x-<name>`.
* `summary`: either a short string or an object with domain counters, e.g., `{ count: 25, open: 6 }`.
* `pagination` (response): `{ "cursor": "abc"?, "next_cursor"?: "xyz", "prev_cursor"?: "pqr", "limit": 50, "order_by": "props.created_at", "order_dir": "asc" }`.
* `ui_hints`: non-semantic renderer hints (see examples below).

### Pagination — Request & Response Contract

**Request (query)**

```
?cursor=<opaque>&limit=50&order_by=props.created_at&order_dir=desc
```

* `limit` default 50, min 1, max 200.
* `order_by` is a dot-path within entity (e.g., `props.priority`).
* `order_dir` in `{asc,desc}`.

**Response (body)**

```json
{
  "pagination": {
    "cursor": "c0",   
    "next_cursor": "c1",
    "prev_cursor": null,
    "limit": 50,
    "order_by": "props.created_at",
    "order_dir": "desc"
  }
}
```

Servers MUST return stable ordering with a tie-break (e.g., `id`) to avoid duplicates/skips across pages. Cursors MUST be opaque and SHOULD remain valid for at least 24 hours.

### Error Envelope — Wire Contract

For non-2xx, return an error envelope with HTTP status aligned to `code`:

```json
{
  "error": {
    "code": "INVALID_PARAMS",
    "message": "status must be one of: open,pending,closed",
    "hints": ["Use 'pending' to triage"],
    "details": { "field": "params.status" }
  }
}
```

**Canonical codes → HTTP:** see [specs/errors.md](./specs/errors.md). Servers MUST map `error.code` to HTTP as specified.

### Actions — Dry Run & Idempotency

* Dry run: `dry_run: true` in body OR `env.dry_run` in runtime. MUST NOT persist. SHOULD return `audit_preview`.
* Commit: omit `dry_run` or set `false`. MUST require `Idempotency-Key` on server.
* Idempotency window: recommended **24h**; duplicate keys MUST return the original committed response body (same status), not re-execute.

### UI Hints (human-only)

```json
"ui_hints": {
  "layout": { "variant": "list" },
  "table": { "columns": ["title", "status", "assignee"] },
  "form": { "layout": "vertical" },
  "deprecated": false
}
```

### Accept vs audience

On endpoints that support both (e.g., runtime `/view`): apply **precedence** defined in README (query > audience > Accept > default).  
`audience` is a delivery-time **visibility filter** over the JACE tree. It MUST NOT be used as an authorization mechanism.

## Discovery

```
GET /.well-known/agent-manifest
```

```json
{
  "version": "0.1.0",
  "protocols": ["JACE/0.1"],
  "dashboard": { "name": "Demo", "base_url": "https://example.com" },
  "views": [{ "name": "tickets.inbox", "href": "/agent/views/tickets/inbox" }],
  "actions": [{ "name": "ticket.update", "method": "PATCH", "href": "/agent/tickets/{id}" }],
  "events": { "sse": "/agent/events", "types": ["ticket.created", "ticket.updated"] },
  "schemas": {
    "jace-view": "/schemas/jace-view.schema.json",
    "jace-view-agent": "/schemas/jace-view.agent.schema.json",
    "jace-action": "/schemas/jace-action.schema.json",
    "jace-action-result": "/schemas/jace-action-result.schema.json",
    "jace-event": "/schemas/jace-event.schema.json"
  }
}
```

## Serve JACE & HTML (express pseudo)

```js
app.get("/agent/views/tickets/inbox", (req, res) => {
  const jace = loadFromDb(); // internal JACE (may include presentation)
  res.setHeader("Vary", "Accept");
  if ((req.headers.accept || "").includes("application/jace+json") || req.query.format === "jace") {
    return res.type("application/jace+json").json(toAgentPayload(jace)); // strip presentation
  }
  return res.type("html").send(renderHtml(jace)); // map semantics to ARIA/HTML
});
```

### Audience filtering
Servers should filter views per audience before sending:

```js
function toAudiencePayload(node, target /* 'agent' | 'human' */) {
  const aud = node.audience ?? 'both';
  if (aud !== 'both' && aud !== target) return null;

  // For agents, also strip presentation
  const omit = target === 'agent' ? ['presentation'] : [];
  const out = deepOmit(node, omit);

  if (Array.isArray(node.children)) {
    out.children = node.children
      .map(ch => toAudiencePayload(ch, target))
      .filter(Boolean);
  }
  return out;
}
```
Use it with content negotiation precedence (query `?format=` / `audience=` > `Accept:`).
