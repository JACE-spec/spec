# Pagination — Canonical Contract

**Requests:** `cursor`, `limit`, `order_by`, `order_dir`.
**Responses:** include `pagination` with both `cursor` and `next_cursor` when applicable. Implement stable sort with `(order_by, id)`.

Agents SHOULD keep `cursor` per-view to resume. Servers MAY include `links` with `rel=next|prev`. Cursors MUST be opaque and SHOULD remain valid for at least **24h**.

## Example

**Request**
```
GET /agent/views/tickets/inbox?limit=50&order_by=props.created_at&order_dir=desc&cursor=c0
```

**Response**
```json
{
  "items": [/* ... */],
  "pagination": {
    "cursor": "c0",
    "next_cursor": "c1",
    "prev_cursor": null,
    "limit": 50,
    "order_by": "props.created_at",
    "order_dir": "desc"
  },
  "links": [
    { "rel": "next", "href": "/agent/views/tickets/inbox?cursor=c1&limit=50&order_by=props.created_at&order_dir=desc" }
  ]
}
```

**Stability**
Servers MUST implement a stable sort using `(order_by, id)` as a tiebreaker to avoid dupes/skips across pages.
