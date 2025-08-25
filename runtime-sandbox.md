## JACE Sandbox Runtime (QuickJS)

**Purpose:** Run **Agent Apps** and **Agent-First Dashboards** safely, deterministically, and with explicit **affordances** (actions).
Agents fetch an JACE **view**, then execute **actions** (`/press` or `/actions`) with `dry_run → commit`, audit trails, and optional sessions.

---

### 1) Architecture

* **QuickJS VM** runs an unbundled JS “module” that sets:

  ```js
  globalThis.jace = {
    manifest: { name, version },
    init(env)  -> state,
    view(state)-> jaceView,
    actions: { [actionName]: (state, params, env) -> ActionResult }
  }
  ```
* **Deterministic runtime**: patched `Date.now()` & `new Date()`, seeded `Math.random`, no timers, no network.
* **HTTP wrapper** exposes JSON endpoints so any agent or client can interact.

---

### 2) Module ABI

```ts
type ActionEnv = {
  session_id?: string;   // for multi-tenant state
  dry_run?: boolean;     // preview-only
  intent?: string;       // human-readable reason
  now?: string;          // optional override (ISO)
  [capability: string]: unknown; // host-injected caps (future)
};

type ActionResult = {
  state: any;                  // next state (runtime persists unless dry_run)
  result?: any;                // domain-level result payload
  audit_preview?: string;      // e.g., "PUSH 7", "ADD 2+3 -> 5"
  events?: any[];              // domain events (optional)
  error?: { code: string; message?: string };
};

globalThis.jace = {
  manifest: { name: string, version: string },

  init(env: ActionEnv): any,            // return initial state (plain object)
  view(state: any): jaceView,            // render JACE JSON (may include presentation)
  actions: {
    [action: string]: (state: any, params: any, env: ActionEnv) => ActionResult
  }
};
```

**Guidelines**

* **Pure-ish actions:** avoid hidden IO; let the host inject capabilities explicitly later.
* **Always set `audit_preview`**; it’s how humans/agents verify intent.
* **Be deterministic**; rely on provided `env.now` or the runtime’s fixed clock.

---

### 3) HTTP API

All requests/responses are JSON.

| Method | Path         | Body                                                  | Returns                                                    |
| -----: | ------------ | ----------------------------------------------------- | ---------------------------------------------------------- |
|    GET | `/healthz`   | —                                                     | `{ ok: true }`                                             |
|   POST | `/load`      | `{ source: string, env? }`                            | `{ ok: true }`                                             |
|   POST | `/load-file` | `{ path: string, env? }`                              | `{ ok: true }`                                             |
|   POST | `/init`      | `{ env? }`                                            | `{ state }`                                                |
|   POST | `/view`      | `{ state? , audience?: "agent" \| "human" }`          | `<html>` \| `{ jace }`                                      |
|   POST | `/press`     | `{ action, params?, env? }`                           | `{ state, jace, result?, audit_preview?, events?, error? }` |
|   POST | `/actions`   | `{ action, params?, dry_run?, intent?, session_id? }` | JACE Action Result                                          |

**Notes**

* `/press` is your minimal shape. `/actions` accepts the **jace envelope** (recommended for agents).
* If `env.dry_run === true` **or** `dry_run: true`, the runtime **does not persist** the returned state.
* If `audience: "agent"` on `/view`, the runtime should **strip `presentation`** fields.
* When `audience: "agent"`, the runtime strips `presentation` and filters out nodes with `audience: "human"`.
* When `audience: "human"`, it may ignore agent-only nodes and keep `presentation`.
* If `audience` is omitted, the runtime chooses representation by `Accept` (`application/jace+json` → JACE JSON, `text/html` → HTML).

#### Endpoint Consistency

- Prefer `/agent/events` for SSE in all examples.
- `/actions` accepts the JACE envelope; `/press` remains minimal for demos.

---

### 4) Determinism & Safety

* **Clock**: `Date.now()` and zero-arg `new Date()` return a fixed timestamp; override with `env.now` if needed.
* **RNG**: `Math.random()` is seeded (xorshift).
* **No network/threads/timers** inside VM.
* **Optional fuel/timeout**: use QuickJS interrupt handler to cap long-running actions.
* **Marshalling**: host↔VM via JSON only (no references).

#### Timeouts & Fuel

- Default max action wall-time: **2s**; on breach, return `{ error: { code: "INTERNAL", message: "Action timed out" } }`.
- Optional instruction-count (fuel) guard; on breach, set message `"Fuel exhausted"`.

---

### 5) Sessions (multi-tenant)

Use `session_id` to isolate state per agent or conversation:

1. Client chooses/creates a `session_id`.
2. On `/view` & `/press`:

   * Host looks up `STATE[session_id]`.
   * Before running, **assign** it to `jace.___state`.
   * After action: if not `dry_run`, **persist** the new state back to `STATE[session_id]`.
3. Return `{ state, jace }` as usual.

> This makes the runtime safe for many concurrent agent users.

**Idempotency:** For commits, the host MUST enforce `Idempotency-Key` de-duplication **per `session_id`**. Dry-run MUST NOT persist state nor consume idempotency keys.

---

### 6) Action semantics

* **Dry-run first**: preview audit & result; do **not** persist.
* **Commit**: repeat without `dry_run` to persist.
* **Idempotency**: accept `Idempotency-Key` on non-dry-run; de-dupe duplicates.
* **Errors**: return `{ error: { code, message }, audit_preview }` with 4xx/5xx status.

---

### 7) Calculator example

**Load & run**
```bash
# Start with embedded demo
npx ts-node src/jace-sandbox-runtime.ts

# Or start with the external module
npx ts-node src/jace-sandbox-runtime.ts src/calc-module.js
```

**Press flow (stack-like)**
```bash
curl -s -X POST http://127.0.0.1:3323/press   -H 'content-type: application/json'   -d '{"action":"calc.input","params":{"digit":7}}' | jq

curl -s -X POST http://127.0.0.1:3323/press   -H 'content-type: application/json'   -d '{"action":"calc.op","params":{"op":"divide"}}' | jq

curl -s -X POST http://127.0.0.1:3323/press   -H 'content-type: application/json'   -d '{"action":"calc.input","params":{"digit":2}}' | jq

curl -s -X POST http://127.0.0.1:3323/press   -H 'content-type: application/json'   -d '{"action":"calc.equals"}' | jq
```

**Agent envelope (recommended)**
```bash
curl -s -X POST http://127.0.0.1:3323/actions   -H 'content-type: application/json'   -H "Idempotency-Key: $(uuidgen)"   -d '{"action":"calc.input","params":{"digit":7},"dry_run":true,"intent":"enter digit"}' | jq
```

**TypeScript client**
```ts
const j = await fetch("http://127.0.0.1:3323/actions", {
  method: "POST",
  headers: { "content-type": "application/json", "Idempotency-Key": crypto.randomUUID() },
  body: JSON.stringify({ action: "calc.equals", dry_run: true, session_id: "sess_1" })
});
const res = await j.json(); // { ok?, state, result?, audit_preview?, ... }
```

---

### 8) Agent-first Dashboard pattern

1. Model your dashboard UI as JACE **views** with **affordances**.
2. Expose those modules through this runtime (or compile your code into JACE).
3. Agents call `/view` (audience=agent) to understand entities, then `/actions` to act.
4. Humans can render the same JACE to HTML — shared semantics, different audience.

---

### 9) Events (optional)

If an action returns `events`, relay them via SSE:

* **Server**: `/events` streams `data: {type,id,ts,entity,changes}\n\n`.
* **Client**: `new EventSource("/events")` → react to updates.

This pairs well with long-running workflows and approval steps.

---

### 10) Roadmap (runtime)

* **Done**: isolated QuickJS VM, deterministic clock/RNG, minimal HTTP API, calc module, TS client.
* **Next**:
  * Dry-run persistence guard (runtime-level)
  * `/actions` endpoint with JACE envelope + Action Result
  * Session store (host map / Redis / KV)
  * Presentation stripping on `audience=agent`
  * Interrupt/fuel guard & timeouts
  * SSE relay for `events`
  * Capability injection (opt-in IO like KV/crypto/clock)

---

### 11) Security checklist

* Never enable `eval`, timers, or network inside the VM.
* Enforce **max body size** (already done) and **timeouts** per call.
* Validate payloads with JSON Schema (Ajv) if user-facing.
* Treat module code as untrusted unless you own it; review before loading.
* Log `action`, `params`, `audit_preview`, `session_id`, `Idempotency-Key`.

---

### 12) FAQ

**Why QuickJS?**
Tiny, deterministic, embeddable — perfect for safe Agent Apps.

**Why actions instead of internal JS calls?**
Affordances create an explicit, auditable machine interface for agents.

**How do I connect this to my real data?**
Inject capabilities via `env` (e.g., `kv`, `rpc`) or wrap the runtime behind a server that translates actions to backend calls.
