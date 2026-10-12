# Deck Plan — Designing the Gateway Layer in a Microservices Architecture

## Talk Details

- **Event:** Virtual tech meetup — Bincom Dev Center, Lagos
- **Speaker:** David Njoagwuani, Senior Software Engineer at CentricAI
- **Time slot:** 30 min total → ~25 min talk + ~5 min Q&A (time slot still to be confirmed)
- **Audience:** Mixed, likely beginner to intermediate developers (students, bootcamp graduates, early/mid-career engineers)
- **Format:** Diagrams plus screenshots of Azure API Management (APIM). No live demo.
- **Platform featured:** Azure API Management (APIM)
- **Branding:** None required. Use a clean, neutral template.
- **Goal:** Attendees leave knowing *what* belongs in the gateway layer, *which* topology fits their system, and *what mistakes* to avoid, with Azure APIM as the concrete example throughout.

## Style Guidelines

- **Teach from first principles.** Define every term (north–south, BFF, policy, etc.) before using it, because some attendees may be new to microservices.
- **Estate-gate analogy for the intuition.** Used in Part 2: one entrance, security checks visitors, directs them to the right house, keeps a logbook, and limits how many enter at once.
- **Food delivery app as the running technical example.** Introduced in Part 2, reused in Parts 3–5, and brought together in Part 7. That way the closing example is familiar rather than new.
- **Concepts first, then Azure.** Explain each idea vendor-neutrally with a diagram, then show how APIM implements it with a screenshot.
- **Keep text light.** One idea per slide, a diagram or screenshot wherever possible, and the details in speaker notes.

---

## Speaker Bio

Full version, for the meetup listing and the speaker's introduction:

> David Njoagwuani is a Senior Software Engineer at CentricAI with 7+ years of experience building backend systems and APIs with Python, Django, and FastAPI. He has worked extensively with microservices, AWS, Azure, Kubernetes, messaging systems, databases, and CI/CD, and currently works on AI and data-intensive platforms. He is particularly interested in backend architecture and the practical challenges of building distributed systems at scale.

Slide version, condensed for the "About me" slide:

- Senior Software Engineer @ CentricAI
- 7+ years building backend systems & APIs (Python, Django, FastAPI)
- Microservices · AWS · Azure · Kubernetes · Messaging · CI/CD
- Currently: AI & data-intensive platforms
- Interested in backend architecture & distributed systems at scale

---

## Running Example — MyFood App

**MyFood** — a fictional Lagos food-delivery platform.

**Clients**
- **Customer mobile app:** browse restaurants, place orders, track delivery
- **Restaurant web dashboard:** manage menus and incoming orders
- **Rider app:** accept deliveries, share live location
- **Partners:** payment provider webhooks, third-party restaurant POS integrations

**Services**
- Users/Auth
- Restaurants & Menu
- Orders
- Payments
- Delivery/Rider Tracking
- Notifications

**Where it shows up**

| Concept | How the example shows it |
|---|---|
| Problem without a gateway (Part 2) | The customer app calls 6 services directly |
| Routing & aggregation (Part 3) | The home screen gets restaurants, promos and active order status in one call |
| Security (Part 3) | Customers, riders and restaurants each sign in differently; partners use subscription keys |
| Rate limiting (Part 3) | Partner POS integrations get throttled per key |
| BFF (Part 4) | Separate gateways for the customer app, restaurant dashboard and rider app |
| Versioning (Part 5) | Old mobile app versions stay in use, so `v1` and `v2` of the Orders API run side by side |

---

## Deck Outline

### Part 1 — Opening (~2 min)

1. **Title slide:** talk title, name, role, Bincom Dev Center meetup, date
2. **About me + agenda:** slide-version bio (above), then the roadmap for the talk

### Part 2 — Why a Gateway Layer? (~4 min)

3. **The problem without a gateway:** the food delivery app's clients call services directly. That makes clients chatty, couples them to the internal topology, and duplicates auth, rate limiting and logging in every service. Introduce the estate-gate analogy here.
4. **What an API gateway is (and isn't):** a single entry point for clients. Contrast it with a load balancer, reverse proxy and service mesh. In Azure terms: APIM vs Application Gateway vs Front Door.
5. **Where the gateway sits:** edge (north–south) vs internal (east–west) traffic, with a simple placement diagram.

### Part 3 — Core Responsibilities (~5 min)

6. **Routing & transformation:** path- and header-based routing, rewriting requests and responses, and combining several calls into one response. *APIM: backends, `rewrite-uri`, `set-body`*
7. **Security at the edge:** TLS termination, authentication, token validation, passing identity on to services. *APIM: `validate-jwt`, Entra ID, subscription keys*
8. **Traffic control & observability:** rate limits, quotas and timeouts (kept light), plus correlation IDs, logs and metrics. *APIM: `rate-limit-by-key`, `quota`, Application Insights*

### Part 4 — Gateway Topology Patterns (~4 min)

9. **Single gateway:** one entry point for everything. Simple to run, but becomes a bottleneck as the system grows.
10. **Backend for Frontend (BFF):** one gateway per client type (customer app, restaurant dashboard, rider app).
11. **Domain / team gateways:** separate gateways per business domain or per team. *APIM: workspaces, multiple instances, self-hosted gateway*
12. **Comparing the patterns:** side-by-side table and a simple decision guide.

### Part 5 — Key Design Decisions (~5 min)

13. **Keep the gateway thin:** what belongs in the gateway vs in the services ("smart endpoints, dumb pipes").
14. **API versioning & lifecycle:** versioning strategies, contract-first design with OpenAPI, deprecation. *APIM: versions vs revisions, developer portal*
15. **Ownership & configuration as code:** who owns the gateway config, declarative config, avoiding a central-team bottleneck. *APIM: Bicep/Terraform, APIOps*
16. **Choosing a gateway:** build vs buy, a quick look at the options (Kong, Envoy, NGINX, KrakenD, AWS API Gateway, Apigee, APIM), how to evaluate them, and APIM tiers in brief.

### Part 6 — Pitfalls (~2 min)

17. **Anti-patterns:**
    - business logic in the gateway
    - the gateway becoming a new monolith
    - a single point of failure
    - over-aggregation
    - one team owning every route

### Part 7 — Putting It Together (~2 min)

18. **Reference architecture on Azure:** Front Door → APIM → services (AKS / Container Apps / Functions), with Entra ID for identity and Application Insights for monitoring.
19. **MyFood, end to end:** follow one order from the customer app through Front Door and APIM (auth, routing, rate limit) to the Orders, Payments and Delivery services, and back.

### Part 8 — Wrap-up (~1 min + Q&A)

20. **Key takeaways:** 3–5 one-line lessons
21. **Resources, Q&A & contact:** further reading (APIM docs, microservices.io patterns), socials, slides link

---

## APIM Screenshots Needed

Captured from an APIM instance set up with the food delivery app's APIs, so the screenshots match the running example.

| Slide | Screenshot |
|---|---|
| 4 | APIM instance overview page |
| 6 | APIs list showing the food delivery APIs and their backends |
| 6 | Policy editor showing a `rewrite-uri` / `set-body` policy |
| 7 | Policy editor showing a `validate-jwt` policy |
| 7 | Products & subscriptions (partner subscription keys) |
| 8 | Policy editor showing `rate-limit-by-key` / `quota` |
| 8 | Analytics or Application Insights dashboard with request metrics |
| 11 | Policy scopes (All APIs → Product → API → Operation). Workspaces are Premium-only, so they are shown as a diagram instead. |
| 14 | API versions & revisions view (Orders `v1` / `v2`) |
| 14 | Developer portal with API documentation |

**Setup note:** The Developer tier is the cheapest tier that includes the developer portal, versions/revisions and the self-hosted gateway. It can take 30–45 minutes to provision. Delete the instance after capturing the screenshots to save credits.

---

## Deck Content Files

Each part is expanded in its own file under `deck/`:

| File | Covers |
|---|---|
| `deck/part_1_opening.md` | Slides 1–2 |
| `deck/part_2_why_a_gateway.md` | Slides 3–5 |
| `deck/part_3_core_responsibilities.md` | Slides 6–8 |
| `deck/part_4_topology_patterns.md` | Slides 9–12 |
| `deck/part_5_design_decisions.md` | Slides 13–16 |
| `deck/part_6_pitfalls.md` | Slide 17 |
| `deck/part_7_putting_it_together.md` | Slides 18–19 |
| `deck/part_8_wrap_up.md` | Slides 20–21 |

Every slide in those files uses the same structure:

```markdown
## Slide N — <Title>

**On slide:** the text that appears on the slide (keep it short)

**Visual:** the diagram or screenshot to use, described clearly enough to build it

**Speaker notes:** what to say, with timing
```

---

## Open Questions to Refine

- [x] Name for the fictional food delivery app → **MyFood**
- [ ] Re-check the time slot once confirmed and trim or expand Parts 4–5 to fit
- [ ] Socials and links for the final slide (LinkedIn, GitHub, X, slides link)
