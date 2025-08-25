# Copy-Paste Snippets

**Fetch a view (agent)**  
```bash
curl -H "Accept: application/jace+json" https://example.com/agent/views/tickets/inbox
```

**Render HTML (server-side pseudo)**
```js
import { renderHtml } from "../renderers/html/index.js";
const jace = await load();          // internal JACE (may include presentation)
const html = renderHtml(jace);      // humans
```

**Strip for agent**
```js
import { toAgentPayload } from "../renderers/agent/index.js";
const payload = toAgentPayload(jace); // drop presentation/classes/style
```

**Execute action**
```bash
curl -X POST https://example.com/agent/tickets/t_8129   -H "Content-Type: application/json"   -H "Idempotency-Key: $(uuidgen)"   -d '{"action":"ticket.update","params":{"status":"pending"},"dry_run":true}'
```

**Paginated fetch**
```bash
curl -H "Accept: application/jace+json"   "https://example.com/agent/views/tickets/inbox?limit=50&order_by=props.created_at&order_dir=desc"
```

**SSE client (browser)**
```html
<script>
  const ev = new EventSource('/agent/events', { withCredentials: true });
  ev.addEventListener('ticket.updated', e => {
    const data = JSON.parse(e.data);
    console.log('updated', data);
  });
</script>
```

**Manifest link tags**
```html
<link rel="agent-manifest" href="/.well-known/agent-manifest">
<link rel="alternate" type="application/jace+json" href="/page?format=jace">
```

**Audience-targeted fetch (agent)**
```bash
curl -H "Accept: application/jace+json"   -X POST https://example.com/view -d '{"audience":"agent"}'
```

**Filter per audience (server)**
```js
const agentView = toAudiencePayload(jace, 'agent'); // strips human-only nodes + presentation
const humanView = toAudiencePayload(jace, 'human'); // strips agent-only nodes
```
