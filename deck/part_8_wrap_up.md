# Part 8 — Wrap-up (~1 min + ~5 min Q&A)

## Slide 20 — Key Takeaways

**On slide:**

1. **The gateway is your system's front door:** one entry point, shared concerns handled once
2. **Keep it thin:** routing and cross-cutting concerns, not business logic
3. **Match the topology to your clients and teams:** start single, evolve to BFF or domain gateways
4. **Treat gateway config as code**, with clear ownership
5. **Design for failure and change:** redundancy, versioning, observability

**Visual:** Five numbered takeaways, one line each, with the estate-gate icon from the start of the talk.

**Speaker notes (~45s):**
- "If you remember nothing else:"
- "The gateway is the front door. Handle shared concerns there once, not in every service."
- "Keep it thin. Business logic belongs in services."
- "Pick a topology that matches your clients and your teams. Start simple and evolve."
- "Treat gateway config like code, and be clear about who owns what."
- "And design it for failure and change, because everything depends on it."

---

## Slide 21 — Resources, Q&A & Contact

**On slide:**

**Further reading:**
- Azure API Management docs — learn.microsoft.com/azure/api-management
- API gateways in microservices — learn.microsoft.com/azure/architecture/microservices/design/gateway
- Gateway Routing / Aggregation / Offloading patterns — learn.microsoft.com/azure/architecture/patterns
- API Gateway pattern — microservices.io/patterns/apigateway.html
- Backends For Frontends — samnewman.io/patterns/architectural/bff
- Microservices (Fowler & Lewis) — martinfowler.com/articles/microservices.html
- APIOps toolkit — github.com/Azure/apiops

**Thank you! Questions?**

**Contact:** [LinkedIn] · [GitHub] · [X] · Slides: [link]

**Visual:** Large "Questions?" in the centre, resources in a smaller column, contact details + a QR code to the slides at the bottom.

**Speaker notes (~15s, then Q&A):**
- "Thank you, and thanks again to Bincom Dev Center. All the links are on this slide, and the slides will be shared afterwards."
- "Let's take questions."

---

## Backup: Likely Questions

Short answers to have ready. Not shown on slides.

**Q: Does the gateway add latency?**
A little, usually a few milliseconds per request. In return you get fewer client round trips (aggregation), caching and protection. Keep policies light and deploy the gateway in the same region as your services.

**Q: API gateway or service mesh — which do I need?**
They solve different problems. The gateway handles north–south traffic (clients → system), and the mesh handles east–west traffic (service ↔ service). Small systems often need only a gateway.

**Q: Should the gateway do authorisation too?**
Coarse checks, yes: "does this token have the `orders.write` scope?" Fine-grained checks that depend on data stay in the service: "does this customer own this order?"

**Q: Can I use APIM with services outside Azure?**
Yes. Backends can be any reachable URL, and the self-hosted gateway can run in any Kubernetes cluster, on-premises or in another cloud.

**Q: Is the Consumption tier enough to start?**
Often, yes, for small or spiky workloads. It lacks some features (no developer portal; some policies like `rate-limit-by-key` aren't supported), so check the feature comparison against your needs.

**Q: What about GraphQL?**
A GraphQL layer can act as an aggregation layer or a BFF. APIM can also expose and protect GraphQL APIs. It's the same thin-gateway principles: auth and limits at the gateway, resolvers in services.

**Q: How do I avoid the gateway becoming a single point of failure?**
Run multiple instances across availability zones, add health checks, and for critical systems deploy multiple regions behind Front Door. In APIM, that means scale units, availability zones and multi-region on Premium.
