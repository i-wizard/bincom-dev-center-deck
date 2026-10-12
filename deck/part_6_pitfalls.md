# Part 6 — Pitfalls (~2 min)

## Slide 17 — Gateway Anti-Patterns

**On slide:**

| ❌ Anti-pattern | ✅ Fix |
|---|---|
| **Business logic in the gateway** | Move it into a service |
| **The gateway as a new monolith:** one huge config every team edits | Split with BFFs or domain gateways; config as code |
| **Single point of failure** | Multiple instances, availability zones, health checks |
| **Over-aggregation:** the gateway calls 8 services per request | Aggregate only what the client needs; use a BFF service |
| **One team owns every route** | Federated ownership with shared guardrails |
| **Internal traffic through the edge gateway** | East–west traffic goes direct, through a mesh, or via events |

**Visual:** The table, with a small warning icon per row. Optionally, highlight the first two rows as "the most common".

**Speaker notes (~2 min):**
- "Let's go through the mistakes I see most often. Each one has a simple fix."
- **Business logic:** "We covered this already. The MyFood discount rule belongs in a service, not the gateway."
- **New monolith:** "We broke up the monolith into microservices, then rebuilt it as one enormous gateway config that every team must edit and deploy together. Split it up and manage it as code."
- **Single point of failure:** "If every request goes through the gateway, the gateway going down means *everything* is down. Run multiple instances across availability zones. In APIM, add scale units, and on Premium, availability zones and multiple regions."
- **Over-aggregation:** "If one request fans out to eight services, the response is only as fast as the slowest one, and fails if any one fails. Aggregate only what the screen needs, and put complex composition in a BFF service."
- **One team owns every route:** "The gateway team becomes a ticket queue, and every release waits on them. Federate the ownership."
- **Internal traffic through the edge:** "From Part 2: Orders calling Payments shouldn't go out the front gate and back in."
