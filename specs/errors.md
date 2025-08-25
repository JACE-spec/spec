# Error Codes

**Canonical codes → HTTP:**

* `INVALID_PARAMS` → 400
* `UNAUTHORIZED` → 401
* `FORBIDDEN` → 403
* `NOT_FOUND` → 404
* `CONFLICT` → 409
* `PRECONDITION_FAILED` → 412
* `RATE_LIMITED` → 429
* `POLICY_DENIED` → 451
* `INTERNAL` → 500
* `DRY_RUN_UNSUPPORTED` → 501

Servers MUST map `error.code` to HTTP as specified above.

## Error Envelope (Wire)
For non-2xx, return:

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
