# FAQ

**Is JACE a replacement for APIs?**  
No. It’s a **machine-native view layer** that complements APIs with workflow context + explicit actions.

**Can `type` reuse HTML tag names?**  
You can (e.g., `section`, `button`), but **keep it semantic**. Domain types like `ticket`, `product`, `card` are encouraged.

**Do agents ever see `presentation`?**  
**Never.** Strip it at the server boundary.

**How do I model multi-step flows?**  
Use separate views per step, explicit affordances, plus **events** for progress.

**How do I support i18n/RTL?**  
Allow `lang` at page/entity; your renderer sets `lang`/`dir` and applies translations.

**Should I send `Idempotency-Key` for dry runs?**  
Recommended but optional. For **commits**, it is **REQUIRED** on POST/PUT/PATCH. Dry-runs never persist state and MUST NOT consume the key.

**URI Templates vs params?**  
If `href` contains `{name}` tokens, clients SHOULD expand them from `params` and also pass the body/query as usual. Servers MUST accept either approach and resolve duplicates with **URI path taking precedence** over body.

**What is `ui_hints` for?**  
Non-semantic hints for human renderers (tables, forms). Never exposed to agents.

**Can I hide parts of a view from agents or from humans?**  
Yes. Set `audience: "human"` for human-only nodes, or `audience: "agent"` for agent-only nodes. Leave it out (or use `"both"`) to include nodes for both.

**Does `audience` replace authentication/authorization?**  
No. `audience` is a presentational filter. You must still enforce access control on the server and in action handlers.

**What happens if a parent is `human` and a child is `agent`?**  
The child’s explicit `audience` overrides the parent for that subtree.
