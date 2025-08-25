# JACE — JSON Agent Context Envelope

**Machine-first pages. Clean HTML for humans. Safer actions for both.**

Start here:

- [Overview](./docs/overview.md) — what JACE is and why agent-first matters
- [Getting Started](./docs/getting-started.md) — serve JACE to agents, HTML to humans
- [Core Concepts](./docs/concepts.md) — entities, props, affordances, presentation
- [Real-World Usage](./docs/real-world.md) — agents + browsers patterns
- [Runtime / Sandbox](./docs/runtime-sandbox.md) — interact, not just read
- [Security](./docs/security.md) — dry runs, audit, idempotency, scopes
- [FAQ](./docs/faq.md)
- [Roadmap](./docs/roadmap.md)

> Author once in JSON. Serve `application/jace+json` to agents. Render accessible HTML for people. No scraping. Safe actions with `dry_run` + `audit_preview`.

## Versioning & Negotiation

* **Protocols:** `JACE/0.1`. Servers MUST list supported protocols in a manifest `protocols` array (see discovery).
* **Media type:** `application/jace+json`. Servers MAY include a `profile` parameter to indicate the protocol profile, e.g.  
  `Content-Type: application/jace+json;profile="https://example.com/profiles/JACE/0.1"`
* **Client behavior on mismatch:** If a client requires a newer protocol, it SHOULD fail fast with a clear message and link to docs.
* **Server behavior on unknown client:** If clients send `User-Agent` or `X-JACE-Client: <name>/<version>`, servers MAY log it but MUST NOT change semantics.

## Content Negotiation Precedence

When both mechanisms are present:

1. `?format=jace|html` (highest precedence)
2. `audience=agent|human` (when supported on `/view`-style endpoints)
3. `Accept:` header (optionally with `profile=` parameter)
4. Default: `text/html` (human)
