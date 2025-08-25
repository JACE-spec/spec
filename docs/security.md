# Security & Governance

- **No script in JACE**: JACE is data-only (even within `presentation`).  
- **Strip `presentation`** for agent responses (agent wire schema forbids it).  
- **Validation**: schema-validate views/actions/events/results.  
- **Idempotency**: include `Idempotency-Key` on writes.  
- **Two-phase actions**: `dry_run` → show `audit_preview` → commit.  
- **Scopes/capabilities**: tie affordances to roles (e.g., `tickets:write`).  
- **Audit headers**: `X-Agent-Intent`, `X-Agent-Trace`.  
- **Rate limits & approvals**: gate destructive actions.  
- **Accessibility**: renderers map semantics to HTML/ARIA (WCAG).  
- **Events hygiene**: only meaningful domain changes (no analytics/hydration noise).

## Presentation Safety

* **Allowed keys:** `tag`, `variant`, `tokens`, `classes`, `style`.
* **Sanitization:** reject `style` containing `url(`, `expression(`, or `@import`.
* **Whitelist tags:** `div`, `section`, `article`, `button`, `span`, `p`, `ul`, `ol`, `li`.
* Agents NEVER see `presentation`.

## Markdown Safety
Renderers MUST sanitize Markdown output:
- Disallow raw HTML or sanitize it (no `<script>`, `on*=` handlers).
- For links, add `rel="noopener noreferrer"` when `target="_blank"`.
- Resolve images to safe origins or proxy them if needed.

## Headers & Audit

* `X-Agent-Intent`: optional mirror of body `intent` for log pipelines.
* `X-Agent-Trace`: opaque correlation id for distributed tracing.

## Approvals & Rate Limits (reference)

* For sensitive actions, return `error.code = "POLICY_DENIED"` with `hints` pointing to approval URLs.
* Use standard `Retry-After` on `RATE_LIMITED` responses.

## Audience is not auth
`audience` controls *visibility* per delivery target. It MUST NOT be treated as an authorization mechanism.  
Servers must validate credentials, scopes, and policies for all actions regardless of audience filtering.
