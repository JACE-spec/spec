# Roadmap

**0.1 (today)**
- Spec + schemas (view/action/result/event)
- Reference server (content negotiation, CORS)
- HTML renderer (ARIA-lite), client libs (TS/Python)
- Static Playground (JACE Runtime) to load & render JACE

**0.2**
- **Interactive Sandbox**: `dry_run`→`commit`, policy prompts
- Agent-wire schema **forbids `presentation`**
- CLI: `jace serve` for static views + renderer
- Better Markdown + i18n helpers
- Canonical **Error codes** and **Pagination** spec
- ARIA mapping reference table
- Audience filtering (`audience: agent|human|both`) finalized in spec and reference implementations.

**0.3**
- Framework adapters (Next/Remix/Nuxt/SvelteKit)
- Event filters & reconnection
- Access control examples (scopes/approvals)
- LangGraph/CrewAI integrations (auto tools from affordances)

**0.4**
- Validation middleware (Ajv/Zod) with friendly errors
- Advanced layouts (tables/forms) without leaking presentation
- End-to-end demo: goal → affordance selection → dry-run → approval → commit

**1.0**
- Stability guarantees, compatibility suite
- Reference implementations
- Security guide & checklists

**Future**
- Agent SEO (discoverable JACE + actions)
- JSX/templating → JACE IR compilers
- Safer autonomy defaults (policy gates baked-in)
