# Content Negotiation Precedence

1) `?format=jace|html` → explicit
2) `audience=agent|human` (where supported)
3) `Accept: application/jace+json | text/html` (optionally with `profile=…`)
4) default: `text/html`

## Caching & Headers
Servers SHOULD set `Vary: Accept` on views so caches keep HTML and JACE variants separately.

## Audience vs Accept
`audience` is a **node-level filter** (agent/human/both) applied to the JACE tree before delivery.  
It does not replace `Accept`—`Accept` still determines the **representation** (JACE JSON vs HTML).

## Media Type Profile
Servers MAY include `profile` on `Content-Type` / `Accept` to indicate JACE profile, e.g.:  
`application/jace+json;profile="https://example.com/profiles/JACE/0.1"`.
