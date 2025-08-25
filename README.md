# Nomie
Trusted, scoped memory and consent for AI agents — reusable across agents and providers

Nomie is a dynamic memory service designed for AI agents. It allows agents to request, store, and retrieve structured user information with privacy, access control, and normalized data formats. This enables vertical agent builders to create smarter, more personalized experiences without having to reinvent user data handling.

---
## Table of Contents

## Why Nomie?
AI agents are powerful, but they often feel repetitive, fragile, or unsafe because they're stateless relative to user preferences and company policies. Nomie solves two hard problems:

1. Trusted memory — store and reuse user preferences and structured facts in a user-owned, auditable way.
2. Controlled access — give agents only the minimum permissions they need, enforce scope and consent, and provide provenance/attestations for high-trust flows.

This reduces user friction (fewer repeated questions), improves developer ergonomics (structured data + normalization), and makes enterprise deployments safer (scoped access and audit logs).

## What Nomie Provides
* **SSO-like developer experience** (`nomie.init`, `nomie.auth.signIn`) — a few lines of code to hook into the memory service.
* **Discovery & capabilities** so agents can learn what Nomie exposes (preferences, bookings, entitlements, etc.).
* **Flexible memory model**: dynamic, schema-guided, and versioned types with canonical IDs + aliases.
* **Fetch & resolve API** that accepts a requested JSON Schema and returns structured values, confidence scores, RBAC info, and optional attestations.
* **User-mediated** `requestInfo` flows so the service can mediate consent and produce normalized values and audit entries.
* **Attestations & provenance** for values returned by Nomie (cryptographic signing optional) so agents can auto-apply values with confidence.
* **RBAC / per-agent scopes** and an audit trail for reads/writes.
---
## High-level architecture
* **Agent**: a vertical agent (e.g., travel booking) integrates the SDK and asks Nomie for fields it needs.
* **Nomie SDK / MCP**: discovery, OAuth, and RPC calls (fetch/store/subscribe). The manifest describes capabilities and schemas.
* **Nomie Server**: resolves keys (exact, aliases, semantic), enforces RBAC/consent, returns structured values with confidence and provenance, or mediates user requests.
* **User**: signs in and grants scopes; Nomie mediates any interactive requests and stores normalized values for reuse.
MCP (Model Context Protocol) or a compatible manifest approach is recommended for discovery and tool description. That gives agents a standard way to discover what memory capabilities Nomie exposes and how to call them.
---
## Integration & SDK (examples)
### Quick developer snippet
A minimal integration that shows the developer experience:
```ts
import { nomie } from "nomie-sdk";

nomie.init({
  clientId: "YOUR_APP_ID",
  redirectUri: "https://yourapp.com/callback"
});

// Trigger SSO-like flow (opens OAuth popup)
await nomie.auth.signIn();

// Fetch and resolve preferences before autofilling a form field
const schema = {
  type: "object",
  properties: { seat_pref: { type: "string" } },
  required: ["seat_pref"]
};

const res = await nomie.memory.fetchAndResolve(user.email, schema, {
  agentId: "travel-agent-v1",
  minConfidence: 0.8
});

if (res.rbac.allowed && res.status === "ok" && res.entries.length) {
  form.seat = res.entries[0].value;
  showBadge(`From Nomie — ${Math.round(res.entries[0].confidence * 100)}%`);
} else if (!res.rbac.allowed || res.status === "rejected") {
  // Create a mediated request that the user will see (modal, in-app card, etc.)
  const req = await nomie.user.requestInfo(user.email, "travel-agent-v1", "What's your seat preference?", schema);
  // show UI to open req.ui.modalUrl or button label
}
```
### Primary SDK functions
* `nomie.init({ clientId, redirectUri })` — initialize SDK with app credentials.
* `nomie.auth.signIn()` — open OAuth/OIDC flow; returns a user object on success.
* `nomie.memory.fetchAndResolve(userId, requestedSchema, opts)` — the core read API; performs matching and RBAC checks and returns values + confidence.
* `nomie.user.requestInfo(userId, agentId, message, requestedSchema, opts)` — create a mediated user prompt and return metadata (modal URL, requestId).
* `nomie.memory.store(userId, payload)` — optional write API to store values (requires scopes/consent).
---
## Discovery & manifest (MCP-style)
Nomie publishes a manifest agents can fetch. The manifest advertises capabilities, auth requirements, and schema links.
Example (excerpt):
```json
{
  "name": "nomie-memory",
  "version": "2025-08-01",
  "auth": {
    "type": "oauth2",
    "scopes": ["nomie.memory.read", "nomie.memory.write", "nomie.profile"]
  },
  "capabilities": [
    {
      "name": "preferences",
      "schema": "https://nomie.example/schema/preferences.json",
      "types": [
        {
          "id": "https://nomie.example/types/seatPreference",
          "title": "Seat preference",
          "aliases": ["seat_pref", "seat_preference", "seatPreference", "seat"],
          "description": "User's seat preference on flights; enum: [Aisle, Window, Middle]",
          "example": {"seatPreference": "Aisle"}
        }
      ]
    }
  ]
}
```
Agents use this to:
* Display a single “Sign in with Nomie” button and required scopes.
* Discover what Nomie can answer and how to request it.
---
## Field matching & resolution
Because Nomie’s storage is dynamic and agents use their own slot names, Nomie resolves requested schema fields via a layered matching algorithm:
1. Exact key match — highest confidence (e.g., `"seatPreference"` exists).
2. Aliases lookup — match set by manifest aliases (`"seat_pref"`, `"seat"`).
3. Canonical type matching — both sides map to the same canonical URI.
4. Normalized name matching — snake_case ↔ camelCase normalization.
5. Schema inference — matching by types/enums in schema.
6. Semantic matching (embeddings) — fallback fuzzy match; returns cosine-similarity-based confidence.
7. Human-in-loop — if confidence < threshold, agent prompts the user for confirmation or calls `requestInfo`.

The SDK exposes helpers like `fetchAndResolve` which return per-field confidence scores and RBAC status so agents can decide whether to auto-apply, ask for confirmation, or request user input.
---
## Security, privacy, and consent
Nomie is designed around explicit consent and minimal exposure:
* **OAuth/OIDC SSO** with fine-grained per-agent scopes (e.g., nomie.memory.read:preferences).
* **Per-field sensitivity metadata**; sensitive fields require elevated consent and cannot be auto-applied.
* **Audit logs**: read/write entries are logged with agentId, userId, and timestamp.
* **Attestations**: optionally provide signed attestations for returned values so agents can safely auto-apply.
* **Revocation**: users can revoke agent access; subsequent fetches will be rejected.
* **TTL & data minimization**: memory entries include TTLs and can be set to expire automatically.
---
## Key Benefits

### For Agent Builders
Why integrate through Nomie instead of simply asking users directly?
* **Attested values**: values returned by Nomie may carry signed attestations that enable auto-apply flows; unmediated chat answers do not.
* **Normalization**: Nomie returns structured, normalized formats (ISO dates, canonical IDs), saving developers parsing work.
* **Cross-agent reuse**: storing values via Nomie lets other authorized agents reuse information, improving UX across apps.
* **Trust & audit**: enterprise customers prefer auditable reads/writes and scoped access.
* **Conversion & UX**: agents that auto-fill with high-confidence values reduce friction and increase completion rates.

### For Consumers
* **Consent-first UX**: Users control what is shared with which agents.  
* **Privacy-preserving memory**: Data is stored with access rules and auditable logs.  
* **Seamless experience**: Personalized actions without repeatedly re-entering preferences.  
* **Dynamic personalization**: Agents adapt to user preferences collected over time.
---
## Demo
---
### Contributing
Contributions and feedback from agent builders are welcome. Helpful contributions include:
* Example schemas for new capability types (contacts, entitlements, preferences).
* Resolver improvements and embedding matching heuristics.
* Security reviews and tests for RBAC and attestation flows.
* Developer experience improvements (SDKs, quickstarts, sample apps).

Please follow the repository’s contribution guidelines and include tests where appropriate.
---
### License
---
