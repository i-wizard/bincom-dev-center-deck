# Part 5 — Key Design Decisions (~5 min)

## Slide 13 — Keep the Gateway Thin

**On slide:**

> "Smart endpoints, dumb pipes"

| ✅ Belongs in the gateway | ❌ Belongs in the service |
|---|---|
| Routing | Business rules (pricing, discounts) |
| Authentication & TLS | Authorisation decisions on data |
| Rate limits & quotas | Database access |
| Logging & correlation IDs | Order validation |
| CORS, simple caching | Multi-step workflows |
| Light transformation | Heavy data composition |

**The test:** *If this logic changes when the business rules change, it belongs in a service.*

**Visual:** The two-column table, with a thin gateway drawn as a narrow pipe between clients and services.

**Speaker notes (~1.25 min):**
- "The phrase 'smart endpoints, dumb pipes' comes from Martin Fowler and James Lewis's original microservices article. The services hold the intelligence; the gateway just moves requests and enforces shared rules."
- "It's very tempting to add 'just one small rule' to the gateway. For MyFood, someone suggests: 'apply a 10% discount for orders over ₦20,000 in the gateway; it's just a header check.' Don't."
- "Business logic in the gateway is hard to test, hard to find, and owned by nobody. When the marketing team changes the discount, the person who knows the gateway config has to make the change."
- "Use the test on the slide: if it changes when business rules change, it belongs in a service."

---

## Slide 14 — API Versioning & Lifecycle

**On slide:**

**Versioning strategies:**
- **URL path:** `/v1/orders`, `/v2/orders` (most common, easiest to see)
- **Header:** `Api-Version: 2`
- **Query string:** `/orders?api-version=2`

**Contract first:** design the API in **OpenAPI**, then import it into the gateway

**In APIM:**
- **Versions:** for *breaking* changes. Consumers choose `v1` or `v2`.
- **Revisions:** for *non-breaking* changes. Test safely, then make current; roll back if needed.
- **Developer portal:** auto-generated docs for every version

**Deprecation:** announce → monitor usage per version → retire

**Visual:** Left: the MyFood Orders API with `v1` and `v2` side by side, an old phone pointing at `v1` and a new phone at `v2`. Right: **screenshots** of (1) the **APIM versions & revisions** view for the Orders API and (2) the **developer portal** showing the Orders API docs.

**Speaker notes (~1.25 min):**
- "Mobile apps make versioning unavoidable. When MyFood releases app version 3, some customers will still be on version 2 for months. Their app expects the old API."
- "So when Orders needs a breaking change, such as renaming a field, we publish `v2` and keep `v1` running. The gateway routes each request to the right version."
- "URL path versioning is the most common because it's obvious. You can see it in every log line."
- APIM: "APIM separates two ideas. *Versions* are for breaking changes that consumers have to opt into. *Revisions* are for safe, non-breaking changes. You can test a revision on a separate URL, then make it current, and roll back if something goes wrong."
- Contract first: "Design the API in OpenAPI first, then import it into APIM. The developer portal generates documentation automatically, so partners can self-serve."
- Deprecation: "Use analytics to see who's still calling `v1` before you turn it off. Never retire a version blind."

---

## Slide 15 — Ownership & Configuration as Code

**On slide:**

**The problem:** gateway changes made by clicking in a portal
- No review · No history · No rollback · Environments drift apart

**The fix:** treat gateway config as code
- Stored in **Git**, changed via **pull requests**, deployed by **CI/CD**

**Who owns what:**
- **Platform team:** the gateway instance + global policies
- **Service teams:** their API definitions + API-level policies

**In APIM:**
- **Bicep / Terraform:** create the APIM instance and infrastructure
- **APIOps:** extract APIs and policies to Git and publish them through pipelines

**Visual:** A left-to-right pipeline: Developer → Pull request (reviewed) → CI/CD pipeline → APIM Dev → APIM Prod. Under it, two labelled lanes: "Platform team: instance & global policies" and "Service teams: APIs & policies".

**Speaker notes (~1.25 min):**
- "Here's a common story. Someone fixes a production issue by editing a policy directly in the Azure portal. It works. Nobody writes it down. Two months later, the dev and prod gateways behave differently and nobody knows why."
- "Gateway configuration is code. It should live in Git, go through pull requests and be deployed by a pipeline, exactly like your services."
- "Ownership matters as much as tooling. If one central team must approve every route change, the gateway becomes a queue. Let service teams own their APIs, and let the platform team own the guardrails."
- APIM: "For the APIM instance itself, use Bicep or Terraform. For APIs and policies, Microsoft's open-source APIOps toolkit extracts your configuration into Git and publishes it through pipelines."
- "Use the portal to explore and learn, and Git to make real changes."

---

## Slide 16 — Choosing a Gateway

**On slide:**

**Build vs buy:** don't build your own gateway from scratch. Use or extend an existing one.

| Managed (cloud) | Self-hosted / open source |
|---|---|
| Azure API Management | Kong |
| AWS API Gateway | Envoy (and Envoy Gateway) |
| Google Apigee | NGINX · Traefik · KrakenD · Tyk |

**How to choose:**
- Where do your services run?
- Which features do you need (developer portal, monetisation, protocols)?
- Latency & scale requirements
- Cost model: per call vs per instance
- Team skills & vendor lock-in

**APIM tiers (in brief):**
- **Consumption:** serverless, pay per call
- **Developer:** full features for non-production, no SLA
- **Basic / Standard / Premium (+ v2 tiers):** production; Premium adds multi-region, VNet and workspaces

**Visual:** Two-column table of options with logos, the criteria as a checklist, and a small APIM tier ladder from Consumption to Premium.

**Speaker notes (~1.25 min):**
- "First, build vs buy. Writing your own gateway sounds fun until you're maintaining TLS, rate limiting, auth and hot reloads yourself. Use an existing one. Some teams write a small custom BFF service, which is fine, but put a real gateway in front of it."
- "Managed gateways like APIM, AWS API Gateway and Apigee mean less to operate. Self-hosted options like Kong, Envoy and NGINX give more control and run anywhere."
- "Choose based on where your services run, what features you need, your latency needs, the cost model and your team's skills. If you're all-in on Azure, APIM is the natural fit. If you run on Kubernetes across clouds, a self-hosted option may suit you better."
- APIM tiers: "Consumption is serverless and great for small or spiky workloads. Developer is what I used for today's screenshots: everything works, but there's no SLA, so never use it in production. Production uses Basic, Standard or Premium; Premium adds multi-region deployment, private networking and workspaces."
- "Check the Azure pricing page for current details. Tiers and features change."
