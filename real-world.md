# Real-World Usage

## Agents (LLMs, bots, RPA)
1) Fetch a view with `Accept: application/jace+json`.  
2) Read entities to understand context.  
3) Execute **affordances** with `dry_run` first, inspect `audit_preview`, then commit.

## Browsers (humans)
- Render JACE → HTML (map semantics to proper tags/ARIA).
- Use **forms** as no-JS fallback; **fetch**/**JS** for UX.
- Apply visuals from `presentation` only.

## Patterns
- **Support inbox**: tickets as `card`, actions: assign, status update, reply.  
- **E-commerce**: products as entities; price/stock actions.  
- **Admin panels**: replace Selenium scraping with JACE views.  
- **Knowledge/news**: articles as entities; summarize/tag/archive actions.
- **Human-only navigation:** site headers/menus with `audience: "human"` so agents don’t parse decorative layout.
- **Agent-only hints:** machine guidance blocks (e.g., suggested actions, pagination cursors) with `audience: "agent"`.

## Migration playbooks
- **HTML-only → JACE**: keep your UI; add JACE view endpoints + HTML renderer that reads the same JACE.  
- **API-first → JACE**: package workflow-oriented views around your API; agents get context + actions.  
- **Iterative**: start with your busiest view and 2–3 high-value affordances.

## Pagination Flow (agent)

1. GET inbox: `?limit=50&order_by=props.created_at&order_dir=desc`
2. If `pagination.next_cursor`, repeat with `?cursor=<next_cursor>`.
3. Agents SHOULD store the last `cursor` per view to resume later.

## Events Flow (SSE)

* Connect: `GET /agent/events` with `Accept: text/event-stream`.
* Reconnect with `Last-Event-ID` to resume from the last `event.id`.
* Only domain events (created/updated/deleted), no analytics.
