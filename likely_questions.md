# Likely Audience Questions

Questions below are grouped by topic and roughly follow the talk. They include beginner questions, implementation follow-ups, trade-offs, and questions prompted by the MyFood example.

Answers are written as short talking points you can say out loud. Azure tier and feature details change often, so items marked *(verify)* should be checked against the current APIM docs before the talk.

---

## Gateway fundamentals

**What problem does an API gateway solve that a reverse proxy or load balancer does not?**
A load balancer spreads traffic across copies of a service, and a reverse proxy forwards requests to hidden servers. Neither understands *APIs*. A gateway knows about consumers, tokens, API versions, rate limits per caller, and request shapes. That lets it handle shared concerns like authentication, throttling, versioning and analytics in one place, instead of in every service.

**Is an API gateway just a reverse proxy with extra features, or is there a more precise distinction?**
Technically, yes: a gateway *is* a reverse proxy at its core. The real distinction is that it is application-aware and consumer-aware. It makes decisions based on who is calling and which API they're calling. Most products also add API management features on top: a developer portal, subscriptions, analytics and lifecycle management.

**When does a system actually need an API gateway? Is it useful before adopting microservices?**
You need one when you have several services behind one public API, several client types, external partners, or the same auth, rate-limiting and logging code repeated in multiple places. It is useful before microservices too. API management features (keys, rate limits, docs, analytics) help a single backend, and a gateway makes later migration easier.

**Could a small application use a gateway in front of a monolith?**
Yes, and it's a good migration strategy. Put the gateway in front of the monolith, then move routes to new services one at a time without clients noticing. This is the *strangler fig* pattern. For a tiny app with one client and no partners, though, it can be unnecessary overhead.

**Does a gateway remove coupling, or does it just move the coupling from clients to the gateway configuration?**
It moves it, but to a much better place. Gateway config is centralised, owned by your team, and can be changed without shipping a new mobile app. The risk is that the gateway becomes a coupling hotspot, which is why it must stay thin and have clear ownership.

**What does "single entry point" mean if the gateway itself has multiple instances or regions?**
It's a *logical* single entry point: one hostname and one API contract, such as `api.myfood.ng`. Physically, that hostname is served by many gateway instances across availability zones or regions, behind DNS or Front Door. Clients see one door; behind it there are several identical doors.

**What is the difference between an API gateway and a service mesh?**
A gateway handles traffic coming *into* the system from clients (north–south). A service mesh like Istio or Linkerd handles traffic *between* services (east–west), usually with a sidecar or node proxy next to each service. The mesh provides mTLS, retries and observability between services.

**Can a service mesh replace the edge gateway, or do they solve different traffic problems?**
They solve different problems. Meshes often have an ingress component, but they generally lack consumer-facing features like subscription keys, a developer portal, per-partner quotas and API versioning. Many systems use both. Small systems often need only the gateway.

**What do north–south and east–west traffic mean in a real deployment?**
North–south: a customer's phone calling `POST /orders` over the internet into your system. East–west: the Orders service calling Payments, or publishing an event to Service Bus, entirely inside your private network.

**Why shouldn't internal service calls go back through the edge gateway?**
It adds latency and cost, makes every internal call depend on the gateway's availability, uses up public rate limits, and can expose internal APIs publicly. Internal calls should stay internal.

**When should services call each other directly, use a service mesh, or communicate through a message broker?**
Call directly when you need an answer right now and have a small number of services. Add a service mesh when you have many services and want consistent mTLS, retries and tracing without writing that code in each one. Use a broker (Service Bus, RabbitMQ, Kafka) when the caller doesn't need to wait. For example, "order placed, now notify the customer" doesn't need to block the order.

**How do Front Door, Application Gateway, and API Management differ, and when would a system use more than one?**
Front Door is a global entry point with a CDN, WAF and multi-region routing. Application Gateway is a regional layer-7 load balancer with a WAF. APIM is the API gateway and management layer. A common combination is Front Door (global + WAF) → APIM (API concerns). Some teams use Application Gateway in front of an internal APIM when they need a regional WAF in a private network.

**Is Front Door required in front of APIM, or is it optional?**
Optional. APIM can be public on its own. You add Front Door for a WAF, DDoS protection at the edge, multi-region failover, or caching of static content. APIM itself has no built-in WAF.

**How do these gateway concepts apply outside Azure?**
All the principles carry over. AWS: CloudFront + WAF → API Gateway. Google Cloud: Cloud Load Balancing + Cloud Armor → Apigee. Anywhere: Kong, Envoy or NGINX. The vocabulary changes ("policies" vs "plugins" vs "filters"), but routing, edge auth, rate limiting, BFFs and the thin-gateway rule are the same.

---

## Gateway responsibilities and request flow

**Which responsibilities should always be handled at the gateway, and which should stay in services?**
At the gateway: TLS, authentication, rate limits and quotas, routing, CORS, correlation IDs and request logging. In services: business rules, data-dependent authorisation, database access, validation that depends on business state, and workflows. The test: if it changes when business rules change, it belongs in a service.

**What is an APIM policy, and when does it run in the request lifecycle?**
A policy is a piece of XML configuration that runs on the request or response. There are four sections:
- `inbound`: before the request reaches the backend
- `backend`: around forwarding the request
- `outbound`: on the response coming back
- `on-error`: when something fails

Policies can use small C# expressions like `@(context.Subscription.Id)` for dynamic values.

**How does path-based or header-based routing work?**
The gateway matches the incoming request against rules. Path-based: `/orders/*` goes to the Orders backend. Header-based: if `X-Client: rider-app`, route to the Rider BFF. In APIM, each API has a base path and backend, and the `choose` policy plus `set-backend-service` handle conditional routing.

**How does canary routing work, and how do you decide which requests go to the new version?**
Send a small slice of traffic to the new version, watch error rates and latency, then increase it gradually. You can choose requests:
- **randomly by percentage:** e.g. 5% of all traffic
- **by user or partner ID:** the same user always sees the same version
- **by header:** internal testers opt in

In APIM, use a `choose` policy, or a load-balanced backend pool with weights *(verify tier support)*. Make sure database changes work for both versions.

**What happens if the gateway routes a request to an unavailable backend?**
The gateway returns a 5xx error after a connection failure or timeout. You can improve on that:
- the `retry` policy, for safe requests only
- backend pools with failover
- a circuit breaker on the APIM backend so failing backends stop receiving traffic
- the `on-error` section to return a clean error message instead of internal details

**How much request or response transformation is reasonable before it becomes business logic?**
Reasonable: renaming fields, adding headers, removing internal fields, format conversion, simple filtering. Too much: calculating prices, applying discounts, deciding order status, or anything with if/else logic based on business meaning. If a product manager would describe the rule, it belongs in a service.

**Can the gateway aggregate responses from multiple services? When should aggregation move to a BFF?**
Yes. APIM can call multiple backends with `send-request` and combine the results. Move it to a BFF service when it needs more than two or three calls, error handling per call, data merging logic, or anything you'd want unit tests for. Code is easier to test and debug than XML policies.

**If the gateway aggregates calls, are those calls made in parallel or one after another?**
In APIM, `send-request` calls run one after another by default. Wrap them in a `wait` policy to run them in parallel. In a BFF service, use async calls (e.g. `asyncio.gather` in Python) to run them in parallel.

**How should partial failures be handled when an aggregated response depends on several services?**
Decide per field what's critical. For the MyFood home screen, if Promos fails, still return restaurants and the active order, with promos empty and a flag saying it's degraded. If something essential fails, return an error. Use short timeouts per call so one slow service doesn't delay the whole response.

**Does every request need to pass through the same gateway, including webhooks and partner traffic?**
Not necessarily the same gateway, but every *external* request should pass through *a* gateway. Webhooks from payment providers should go through it for logging and IP filtering, though they often use signature verification rather than JWTs. Partner traffic may get its own APIs, products or even a separate gateway with stricter limits.

**How do you handle large uploads, streaming responses, WebSockets, or long-running requests through a gateway?**
- **Large uploads:** don't stream big files through the gateway. Have the API return a short-lived upload URL (e.g. a Blob Storage SAS URL) and upload directly. This is the *valet key* pattern.
- **Streaming:** APIM can pass responses through without buffering (`buffer-response="false"` on `forward-request`).
- **WebSockets:** APIM supports WebSocket APIs *(verify tier support)*.
- **Long-running work:** return `202 Accepted` with a status URL and let the client poll or receive a notification (*async request-reply*).

Check timeouts at every hop: Front Door, APIM and the backend.

---

## Authentication, authorisation, and security

**What is the difference between authentication and authorisation in the MyFood example?**
Authentication: "Is this really customer Ada?" The gateway checks her token. Authorisation: "Can Ada cancel *order 123*?" Only the Orders service knows whether that order belongs to Ada and whether it's still cancellable.

**If the gateway validates a JWT, why should the downstream service do any identity checks?**
Because the gateway only checks that the token is valid. It doesn't know your data. The service must still check ownership and permissions on the specific resource. Also, defence in depth: if someone reaches the service another way, it shouldn't blindly trust the request.

**Is forwarding a user ID in a header safe? How do services know a client did not forge that header?**
Only if two rules hold:
1. The gateway always *overwrites* the header, using `set-header` with `exists-action="override"`, so a client-supplied value is thrown away.
2. Services are reachable *only* from the gateway (private network).

For stronger guarantees, forward the original token and let the service validate it too, or have the gateway issue a signed internal token.

**Should services validate the token themselves as well, or trust the gateway?**
For sensitive services like Payments, validate in both places. Signature checks are cheap and use cached keys. For low-risk internal services in a locked-down network, trusting the gateway's header is common and acceptable. The more sensitive the data, the less you should rely on the network alone ("zero trust").

**How do you securely restrict direct access to services so callers cannot bypass the gateway?**
- Put services in a private network with no public IPs.
- Use network security groups or firewall rules to accept traffic only from the gateway's subnet.
- Use private endpoints for PaaS services.
- Optionally use mTLS or a shared secret between gateway and backends.

On Azure, run APIM in a VNet so it can reach private backends.

**What happens if the gateway or identity provider is unavailable?**
If the gateway is down, all external traffic stops, which is why you run it highly available. If the identity provider is down, new sign-ins fail, but existing tokens keep working. The gateway validates them using signing keys it has already downloaded and cached.

**How are token expiry, key rotation, issuer validation, and audience validation handled?**
`validate-jwt` handles all four:
- **Expiry:** checked automatically, with a small tolerance for clock differences.
- **Key rotation:** when you point it at the `openid-config` URL, APIM downloads the signing keys and picks up new ones automatically.
- **Issuer and audience:** configured with `<issuers>` and `<audiences>`, so tokens issued for another app or tenant are rejected.

**When should an API use OAuth/JWT authentication versus an APIM subscription key?**
Use OAuth/JWT when you need to know *who the user is*, as with customers, restaurants and riders. Use subscription keys to identify *which application or partner* is calling, for rate limits, quotas and analytics. For partners, combine them: a subscription key plus OAuth client credentials, or mTLS.

**Are subscription keys secrets, and how should partners store and rotate them?**
Yes, treat them like passwords. Partners should keep them in a secret store, never in mobile apps or front-end code. APIM gives each subscription a primary and secondary key, so you can rotate without downtime: switch the partner to the secondary key, regenerate the primary, then switch back.

**Do subscription keys identify a partner, authenticate a user, or both?**
They identify the subscription, which usually means a partner or application. They don't identify an end user. On their own they are weak authentication, because a leaked key works for anyone, which is why sensitive partner APIs add OAuth or mTLS.

**How would customer, restaurant, and rider identities be represented when they use different sign-in flows?**
Use Microsoft Entra External ID for customer-facing sign-in, with separate app registrations (and therefore separate token audiences) for the customer app, restaurant dashboard and rider app. Add app roles or claims for user type. The gateway validates the audience per API, so a customer token can't call rider endpoints. Internal staff tools would use the regular workforce Entra ID tenant.

**Where does TLS terminate, and is traffic encrypted again between the gateway and backend?**
TLS terminates at each hop: Front Door, then APIM. Best practice is to re-encrypt each hop, so Front Door → APIM is HTTPS and APIM → backend is HTTPS. For extra assurance, APIM can present a client certificate to backends (mTLS).

**How should secrets and certificates be managed, for example with Key Vault?**
Store secrets and certificates in Key Vault. APIM reads them through a managed identity: secrets as *named values* that reference Key Vault, and custom domain certificates directly from Key Vault. Rotating them in Key Vault then updates APIM automatically. Never paste secrets into policies.

**What protections are needed beyond JWT validation and rate limiting, such as WAF rules or input validation?**
- **WAF** (Front Door or Application Gateway) for common attacks like SQL injection and XSS.
- **Schema validation:** `validate-content` and `validate-parameters` check requests against the OpenAPI spec.
- **Network rules:** IP filtering for partners and webhooks.
- **Browser access:** strict CORS.
- **Request size limits.**
- **API threat detection:** Microsoft Defender for APIs.

Services must still validate their own inputs.

---

## Rate limits, reliability, and observability

**How do you choose a rate limit or quota for a partner or customer?**
Start from data:
- **Capacity:** what your backend can handle.
- **Expected usage:** what a well-behaved client actually needs.
- **Commercial tier:** what the partner is paying for.

Set limits comfortably above normal usage, monitor how often clients hit them, and adjust. Different products (Free, Standard, Enterprise) can have different limits.

**What is the difference between a rate limit and a quota?**
A rate limit protects against *bursts*: 100 calls per minute. A quota caps *total usage over a long period*: 10,000 calls per day or month. Rate limits protect your system; quotas usually reflect a plan or contract.

**What does a client do after receiving `429 Too Many Requests`?**
Wait and retry. APIM includes a `Retry-After` header saying how long to wait. Clients should respect it and use exponential backoff with some randomness ("jitter") so they don't all retry at the same moment.

**Are APIM rate-limit counters shared across gateway instances and regions?**
Counters are synchronised across the units in one region, but approximately, so limits are not perfectly exact under heavy load. Across multiple regions, counters may be tracked per region *(verify in the current docs for your tier)*. If you need exact global counting, for example for billing, track usage in a dedicated system too.

**How do you avoid one customer or partner exhausting a shared limit?**
Rate-limit *by key*: per subscription, per user ID from the token, or per IP address, using `rate-limit-by-key`. That way each caller has their own bucket. Add a separate, higher overall limit to protect the backend in total, and consider `limit-concurrency` for expensive operations.

**How should timeouts be chosen, and how do they interact with backend timeouts?**
Base them on the backend's real response times (e.g. its 99th percentile) plus a margin. Each hop should have a slightly *shorter* timeout than the hop in front of it: client > Front Door > APIM > backend. Otherwise an outer layer gives up while the inner layer keeps working on a request nobody is waiting for.

**Should the gateway retry failed requests? How do you avoid duplicate orders or payments?**
Only retry operations that are safe to repeat, like `GET` requests. For `POST /orders`, the client sends an `Idempotency-Key` header; the Orders service stores it and returns the original result if the same key arrives again. Payment providers like Paystack and Flutterwave support this with unique transaction references. Never blindly retry payments at the gateway.

**How do circuit breakers and bulkheads fit into this architecture?**
A circuit breaker stops sending traffic to a failing backend for a while, so it can recover and callers fail fast. APIM backends support circuit-breaker rules. A bulkhead isolates resources so one failing part can't sink the rest, like compartments on a ship. Examples: separate limits or backend pools for partners vs customers, or `limit-concurrency` per operation.

**What happens to the whole system if the gateway goes down? How is it made highly available?**
All external traffic stops, so the gateway must be one of the most reliable parts of the system:
- multiple units (instances)
- availability zones
- multi-region deployment for critical systems
- health monitoring
- careful, staged config changes

In APIM, availability zones and multi-region are Premium features *(verify for v2 tiers)*.

**How do availability zones, multiple regions, health checks, and autoscaling help?**
- **Availability zones:** survive a data centre failure within a region.
- **Multiple regions:** survive a whole region failing.
- **Health checks:** let Front Door or a load balancer stop sending traffic to unhealthy instances.
- **Autoscaling:** adds capacity during spikes like Friday-night food orders. APIM supports autoscale on capacity metrics in the classic tiers.

**What is a correlation ID, and how is it propagated across asynchronous events as well as HTTP calls?**
A unique ID attached to a request at the gateway and passed along everywhere it goes, so you can find every log line related to one request. Over HTTP it travels as a header. Over Service Bus, put it in the message's `CorrelationId` or a custom property, and have consumers include it in their logs. OpenTelemetry and Application Insights propagate trace context automatically with the standard `traceparent` header.

**What should be logged at the gateway, and how do you avoid logging tokens or sensitive data?**
Log method, path, status, duration, backend duration, subscription/caller ID, correlation ID and client IP. Avoid `Authorization` headers, subscription keys, passwords, card data and personal data in bodies. APIM's diagnostic settings let you choose which headers and how much of the body to log, and mask sensitive query parameters and headers. Also remember data protection laws like Nigeria's NDPA.

**How do you trace one failed order across APIM, Orders, Payments, and Service Bus?**
Search Application Insights for the correlation or operation ID. The end-to-end transaction view shows the APIM request, the Orders call, the Payments call and the Service Bus message on one timeline, with the failing step highlighted. This only works if every service is instrumented and propagates the ID.

**What is the difference between logs, metrics, and distributed traces in this setup?**
- **Logs:** detailed records of individual events ("order 123 failed: card declined").
- **Metrics:** numbers over time (requests per second, error rate, latency). Good for dashboards and alerts.
- **Traces:** the full path of one request across services, with timings for each step.

Metrics tell you *something* is wrong, traces tell you *where*, and logs tell you *why*.

**Does sending all traffic through a gateway create extra latency or a performance bottleneck?**
It adds a small amount of latency, typically a few milliseconds, more with heavy policies. It becomes a bottleneck only if under-scaled, so monitor APIM's capacity metric and scale out before it's saturated. The trade-off is usually worth it, especially when aggregation saves clients several round trips.

---

## Topology patterns and the BFF question

**What is the difference between a BFF and simply routing a client request directly to a service?**
Routing passes the request through unchanged to one service. A BFF is a backend built for one client. It can call several services, combine and reshape the data, and return exactly what that screen needs.

**Is a BFF one service per client type, or one service per backend service?**
One per client type, or more precisely, one per user experience: customer app, restaurant dashboard, rider app. It is *not* one per backend service. That would just be an extra hop.

**What logic belongs in a BFF, and how is it different from logic in the API gateway or domain services?**
- **Gateway:** cross-cutting concerns for everyone (auth, limits, logging).
- **BFF:** presentation logic for one client (aggregation, shaping, client-specific formatting).
- **Domain services:** business rules and data ("can this order be cancelled?").

A BFF should never contain business rules.

**Does each client need its own BFF, or can some clients share one?**
Clients with similar needs can share. iOS and Android customer apps almost always share one BFF. Split when needs diverge enough that one team's changes keep breaking or slowing down another's.

**How do you keep BFFs from duplicating authentication, validation, and other shared concerns?**
Keep authentication, rate limits and logging at the edge gateway in front of the BFFs. Put remaining shared code (token parsing, HTTP clients, error formats) in a shared library or service template. Keep business validation in domain services, not BFFs.

**What are the operational costs of adding a BFF for every client?**
More deployments, pipelines, monitoring, alerts, on-call responsibility and security patching, plus another network hop and the risk of logic drifting between BFFs. It's worth it only when the client teams and their needs are genuinely different.

**When should a team move from a single gateway to BFFs?**
When clients need very different response shapes, when one client's changes keep affecting another, when the single gateway config is full of client-specific special cases, or when separate client teams want independent release cycles.

**When should it split gateways by domain or team instead?**
When the *teams* are the bottleneck rather than the clients: many teams editing the same gateway config, slow reviews, or release coordination between teams. Domain gateways give each team control over their own APIs.

**Can a system combine an edge gateway, BFFs, and domain-owned APIs? How does a request flow through those layers?**
Yes, and it's common. The customer app calls the edge gateway (auth, WAF, limits), which routes to the Customer BFF. The BFF calls the Orders and Restaurants domain APIs and combines the results, and the response travels back the same way. Each layer has one clear job.

**Does having several gateway layers add too much latency or make debugging harder?**
Each hop adds a few milliseconds and another place for failures, so only add layers that earn their place. Correlation IDs and distributed tracing make debugging manageable. Without them, multiple layers become very painful.

**What does federated gateway ownership look like in practice?**
The platform team owns the APIM instance, networking, global policies (auth, logging, CORS) and the pipelines. Service teams own their API definitions and API-level policies in their own repos or folders, and changes go through pull requests. The platform team reviews only global changes or provides automated checks.

**How can platform teams enforce common security policies while allowing service teams to manage their APIs?**
Put mandatory policies at the global (All APIs) scope. In CI, automatically reject API policies that omit `<base />`, since that would skip global policies, and check for required elements. Use Azure Policy for infrastructure rules. Automated guardrails scale better than manual reviews.

**What are APIM policy scopes, and how do policies at All APIs, Product, API, and Operation levels interact?**
Policies are defined at each scope (plus workspace, on tiers that support it) and combine from broadest to narrowest. The `<base />` element marks where the parent scope's policies run. Put it first to run the parent's policies before your own, later to run yours first, or omit it to *override* the parent. Omitting it is a common source of accidentally skipped security.

**What are APIM workspaces, and which teams or tiers can use them?**
Workspaces let separate teams manage their own APIs, products and policies inside a shared APIM instance, with access control so teams can't touch each other's work. They're designed for federated API management and available on the Premium tier *(verify current tier availability, including v2)*.

**What does the APIM self-hosted gateway do, and when is it useful to run it near services?**
It's the APIM gateway packaged as a container that you run yourself, in Kubernetes or Docker, on-premises or in another cloud, while managing it from Azure. Use it when services run outside Azure, when you need low latency inside the same cluster, or for data residency reasons. It needs outbound connectivity to Azure for configuration and is available on Developer and Premium tiers.

**How do you avoid inconsistent standards when teams own separate gateways?**
Use shared standards backed by automation: API design guidelines, an OpenAPI linter (e.g. Spectral) in CI, mandatory global policies, shared policy fragments, and templates for new APIs. A shared catalogue such as Azure API Center also helps with discovery and governance.

**How does Conway's Law influence the way gateway ownership should be organised?**
Conway's Law says systems end up mirroring the communication structure of the teams that build them. If the gateway is owned by one central team but APIs are owned by many teams, the gateway becomes a queue. Align gateway ownership with team boundaries: domain teams own domain APIs, and the platform team owns shared infrastructure.

---

## API design, versioning, and lifecycle

**Which API versioning strategy should a team choose: URL path, header, or query string?**
For public and mobile APIs, URL path (`/v1/orders`) is the most common choice: it's visible in logs, easy to test in a browser and easy to route. Headers keep URLs clean but are less visible. Query strings are used by some platforms, including Azure's own APIs. Consistency across your APIs matters more than which option you pick.

**Are breaking changes the only reason to create a new API version?**
Mostly, yes. Additive changes like new optional fields or new endpoints don't need a new version. Other reasons: a major redesign of the resource model, a change in meaning or behaviour even when the shape stays the same, or a security or auth model change that old clients can't follow.

**What is the difference between an APIM version and a revision?**
A *version* is a separate, visible API that consumers choose, like `v1` or `v2`. Use it for breaking changes. A *revision* is a change within one version, for non-breaking changes. You can test a revision on its own URL, make it current when ready, and roll back. Revisions also keep a change log.

**How long should an older API version remain available?**
It depends on who uses it. For mobile apps, keep it until usage drops to an acceptable level, often tied to a minimum supported app version and a force-update prompt. For external partners, follow what you promised, commonly 6–12 months' notice. Decide based on data, not a fixed date alone.

**How do you identify which clients still use a version before retiring it?**
Use APIM analytics or Application Insights, broken down by API version and subscription (for partners). For mobile, have the app send its version in a header, such as `X-App-Version`, and log it. Then you know exactly who's left and can contact partners directly.

**What does a safe deprecation process look like for mobile apps and external partners?**
1. Announce it with a date.
2. Add `Deprecation` and `Sunset` response headers.
3. Update the developer portal.
4. Monitor usage, and contact remaining partners directly.
5. For mobile, prompt or force app updates.
6. Run "brownouts" (brief planned outages of the old version) to flush out forgotten callers.
7. Retire the version.

**How do OpenAPI contracts fit into API-first development and APIM?**
Teams agree on the OpenAPI contract first, then build the backend and client in parallel against it. Import the spec into APIM to create the API, operations and developer-portal docs automatically. Frameworks like FastAPI generate OpenAPI from code, which works well if you treat the generated spec as a reviewed artefact.

**What happens if the OpenAPI document and deployed backend behavior drift apart?**
Docs lie, clients break, and validation may reject valid requests or let invalid ones through. Prevent it by generating the spec from code or code from the spec, running contract tests in CI (e.g. Schemathesis or Pact), and publishing the spec to APIM from the same pipeline that deploys the backend.

**How can partners test an API before production access?**
Give them a sandbox environment with test data and a separate sandbox product and keys. Use APIM's `mock-response` policy for APIs not yet built, and the developer portal's "Try it" console for exploring. Grant production keys only after they pass integration testing.

**What role does the APIM developer portal play, and can it be used for private APIs?**
It's a self-service website where consumers discover APIs, read docs, try requests and get subscription keys. Yes, it works for private or internal APIs: you control visibility by user groups and products, can require sign-in, and can disable or customise it.

**How should APIs handle backward-compatible additions such as optional fields?**
Add fields as optional, never change the type or meaning of existing fields, and never remove fields within a version. Clients should follow the "tolerant reader" rule: ignore fields they don't recognise. Then additions don't need a new version.

---

## Ownership, deployment, and choosing a product

**Why is editing gateway configuration directly in the portal risky?**
There's no review, no history, and no easy rollback, and environments drift apart because dev and prod no longer match. The next automated deployment may silently overwrite the manual fix. Use the portal to explore and Git to make real changes.

**What should be managed with Bicep or Terraform, and what should be managed with APIOps?**
Bicep or Terraform: the APIM instance, its tier and scaling, networking, managed identity, Key Vault access, custom domains, and logging setup. APIOps: the APIs, operations, policies, products, named values and backends, which change much more often and are owned by API teams.

**How do teams promote gateway configuration safely from development to staging and production?**
Store the config in Git, deploy it to dev automatically, run tests, then promote the *same* config to staging and production. Use per-environment overrides for things that differ, like backend URLs and named values. APIOps supports these overrides. Gate production behind an approval step.

**How do pull requests and automated checks catch invalid policies before deployment?**
In CI:
- check the XML is well-formed
- lint OpenAPI specs
- check required policy elements (`<base />`, auth)
- reject secrets in config

Then deploy to a dev APIM instance and run smoke tests. Some policy errors only appear when APIM accepts or rejects the deployment, so that dev deployment step matters.

**Who should own global policies, API definitions, and backend service changes?**
Global policies: the platform or security team. API definitions and API-level policies: the team that owns that service. Backend service changes: the service team, coordinated with their API definition in the same pipeline where possible.

**How can configuration-as-code support rollback if a policy change breaks production traffic?**
Revert the commit and redeploy the previous known-good configuration through the same pipeline. For API changes, APIM revisions allow switching back to the previous revision quickly. Fast rollback depends on a fast pipeline, so test it before you need it.

**Should a team build a gateway, use a managed service, or run an open-source gateway?**
Almost never build one from scratch. Use a managed service if you're on one cloud and want little operational work. Use an open-source gateway (Kong, Envoy-based, NGINX) if you need portability, run on Kubernetes across environments, or want full control and have the skills to operate it.

**How do APIM, AWS API Gateway, Apigee, Kong, Envoy, and NGINX compare for a particular use case?**
- **APIM:** Azure-native API management with developer portal, products, policies and a self-hosted option.
- **AWS API Gateway:** serverless, pay-per-request, tight Lambda integration, lighter on API management features.
- **Apigee:** enterprise API management with strong analytics and monetisation, on Google Cloud.
- **Kong:** popular open-source gateway with a plugin ecosystem; runs anywhere, with an enterprise edition.
- **Envoy:** high-performance proxy behind many meshes and gateways, powerful but lower-level.
- **NGINX:** fast, simple reverse proxy; API features require config or paid add-ons.

**What criteria matter most when choosing a gateway: cost, latency, protocols, developer portal, portability, or team skills?**
Start with where your services run and what your team can operate. Those rule options out fastest. Then features you actually need (partner portal? GraphQL? WebSockets?), cost at your expected traffic, and latency. Portability matters most if you're genuinely multi-cloud or on-premises.

**How should vendor lock-in affect the choice?**
Be aware of it, but don't over-weight it. Gateway lock-in is mostly in configuration (policies, products), which is real but migratable. Reduce it by keeping the gateway thin, using OpenAPI for contracts and standards like OAuth and OpenTelemetry. Thin gateways are much easier to replace.

**Which APIM tier is appropriate for development versus production?**
Development: Developer (full features, no SLA, cheap) or Consumption for simple cases. Production: Basic/Standard (classic or v2) for most workloads, and Premium for multi-region, availability zones, full VNet isolation, workspaces or self-hosted gateways. Never run production on Developer.

**What are the practical differences among Consumption, Developer, Basic, Standard, Premium, and the v2 tiers?**
- **Consumption:** serverless, pay per call, scales automatically. Possible cold starts, no developer portal, some policies unavailable.
- **Developer:** all features for non-production, single unit, no SLA.
- **Basic / Standard:** production SLA, scale by adding units, limited networking.
- **Premium:** multi-region, availability zones, VNet injection, workspaces, self-hosted gateway.
- **v2 tiers (Basic v2, Standard v2, Premium v2):** newer platform, provisions in minutes rather than up to an hour, with improved networking options.

Check the feature comparison table *(verify)*.

**How do APIM pricing and capacity change as traffic grows?**
Consumption scales automatically and charges per call, which is cheap at low volume but grows with usage. The other tiers charge per unit per hour. Each unit handles a certain throughput, and you add units (manually or by autoscale) as the capacity metric rises. At steady high traffic, unit-based tiers usually work out cheaper than per-call pricing.

**Can a small team afford APIM, or is a self-hosted gateway more appropriate?**
Consumption is very affordable at low traffic, and Basic v2 is a reasonable production entry point. A self-hosted open-source gateway is free to license but costs engineering time to run, patch, scale and secure. For a small team already on Azure, managed APIM is usually cheaper overall.

**Which APIM features shown in the talk require particular tiers or deployment modes?** *(verify all)*

| Feature | Availability |
|---|---|
| `rate-limit-by-key`, `quota-by-key` | Not in Consumption |
| Developer portal | Not in Consumption |
| Workspaces | Premium |
| Self-hosted gateway | Developer and Premium |
| Availability zones & multi-region | Premium |
| Full VNet injection | Developer and Premium; v2 tiers have their own VNet options |
| Versions, revisions, `validate-jwt`, products | All tiers |

---

## MyFood reference architecture and walkthrough

**Why does the reference architecture put Front Door before APIM?**
Front Door adds things APIM doesn't have: a WAF against common attacks, edge DDoS protection, a global network close to users, and automatic failover between regions. APIM then focuses on API concerns. It's the estate's outer wall and the gatehouse behind it.

**Could APIM be public-facing without Front Door, and what protections would be missing?**
Yes. Many systems start that way. You'd lose the WAF, edge-level attack filtering, multi-region failover and global acceleration. APIM's rate limits and token validation still protect the services, but malicious payloads would reach APIM and possibly your backends unless the services validate input carefully.

**Why are the backend services in a private network, and how does APIM reach them?**
So the only way in is through the gateway; nobody can walk around the gate. APIM is deployed into or integrated with the VNet and calls services on private addresses or private endpoints. Connecting Front Door to an internal APIM needs Private Link or an Application Gateway in between *(verify current support for your tier)*.

**Where would the BFF services run: AKS, Container Apps, Functions, or somewhere else?**
For most teams, Azure Container Apps is the best default: containers without managing Kubernetes, with scaling built in. Use AKS if you already run Kubernetes and have the skills. Functions suit very lightweight or event-driven BFFs but can have cold starts. The choice doesn't change the gateway design.

**Why does Orders call Payments directly while Delivery and Notifications react to an event?**
The customer needs to know *right now* whether payment succeeded before the order is confirmed, so that call is synchronous. Assigning a rider and sending notifications can happen a moment later, and the order shouldn't fail because Notifications is down. So those react to an event asynchronously.

**What happens if the payment succeeds but order creation or event publication fails?**
That's a distributed transaction problem. Two patterns help:
- **Transactional outbox:** Orders saves the order and the event in its database in the same transaction, then a separate process publishes the event, so the event is never lost.
- **Saga with compensation:** if the order can't be completed after payment, trigger a refund.

Idempotency makes the retries safe.

**How do you make payment and order operations idempotent so retries do not charge twice?**
The client generates an idempotency key per order attempt and sends it with `POST /orders`. Orders stores it and returns the original result for any repeat. When calling the payment provider, Orders uses a unique transaction reference (Paystack and Flutterwave support this), so the provider rejects duplicates too.

**Does the customer receive `201 Created` before delivery and notification work is complete?**
Yes. `201` means the order is created and paid. Rider assignment and notifications happen afterwards. The app shows "Finding a rider…" and updates through push notifications, polling or a WebSocket.

**How are Service Bus events secured, retried, and handled when a consumer is unavailable?**
- **Security:** services connect with managed identities and role-based access, ideally through private endpoints, with no connection strings in code.
- **Persistence:** messages wait in the queue or subscription until the consumer comes back (up to their time-to-live).
- **Retries:** failed messages are retried up to a maximum delivery count, then moved to a dead-letter queue for investigation.

**What data belongs in the `OrderPlaced` event, and how do services avoid becoming tightly coupled to its schema?**
Include IDs and the facts consumers commonly need: order ID, customer ID, restaurant ID, delivery address, timestamp and an event version. Don't include everything. To avoid coupling: version the event, only make additive changes, have consumers ignore unknown fields, and document the schema like an API.

**In the end-to-end flow, where are errors translated into client-friendly responses?**
Services return structured errors, ideally the standard "problem details" format (RFC 9457). The BFF translates them into messages that make sense for its client. The gateway's `on-error` section handles errors the gateway itself generates (401, 429, backend unavailable) and hides internal details like stack traces.

**How does the architecture handle a new Orders version while old mobile apps are still active?**
APIM exposes both `v1` and `v2` of the Orders API. Each version either routes to the same service, which supports both contracts, or `v1` routes to a thin adapter that translates to the new model. Usage of `v1` is monitored, and it's retired once old app versions fall below a threshold or are forced to update.

**What would change if MyFood ran on AWS, on-premises, or across multiple clouds?**
Only the product names; the design stays the same.

| Role | AWS | On-premises |
|---|---|---|
| Edge + WAF | CloudFront + AWS WAF | NGINX or HAProxy |
| Gateway | API Gateway | Kong or Envoy |
| Services | ECS, EKS or Lambda | Kubernetes |
| Identity | Cognito | Keycloak |
| Messaging | SQS/SNS or EventBridge | RabbitMQ or Kafka |
| Observability | CloudWatch + X-Ray | Prometheus, Grafana, Jaeger |

For multi-cloud, use a portable gateway like Kong, or APIM's self-hosted gateway in each cloud.

**Is this architecture appropriate for a small product, or is it intended for a larger system?**
It's for a growing, mid-to-large system. For a small product, start with one gateway (or even Consumption-tier APIM) in front of a monolith or a few services. Skip Front Door and BFFs, and add them when real needs appear. Also note there's no Azure region in Nigeria yet; the nearest are South Africa and Europe, so measure latency for Lagos users.

---

## Practical and broader questions

**What is the first gateway change you would make in an existing system with duplicated auth and logging?**
Put a gateway in front with pass-through routing. Clients change one URL; nothing else changes. Then move token validation and correlation IDs to the gateway, and remove the duplicated code from services one at a time. Small, reversible steps.

**What metrics would you use to know whether the gateway is helping or hurting?**
- **Gateway overhead:** total time vs backend time (APIM logs both).
- **Errors:** rates split between gateway and backend errors.
- **Limits:** how often callers hit `429` and `401`.
- **Capacity:** APIM's capacity metric.
- **Team speed:** how long it takes to ship a new API or route change, and the number of incidents caused by gateway config.

**How would you test gateway policies and routing locally without a full Azure deployment?**
Honestly, this is one of APIM's weak points. Options:
- Run the self-hosted gateway in Docker locally. It still needs a connection to an APIM instance for config.
- Use a cheap dev instance with the built-in test console and request tracing.
- Microsoft's policy toolkit lets you write and unit-test policies in C# *(verify current status)*.

Open-source gateways like Kong or Envoy run fully locally in Docker Compose.

**What are common production incidents caused by gateway configuration?**
- expired TLS certificates
- a missing `<base />` that skips global auth
- wrong backend URLs after promoting config between environments
- rate limits set too low for real traffic
- timeout mismatches between layers
- broken CORS after a change
- gateway capacity running out during a traffic spike
- manual portal changes overwritten by a deployment

**How do you balance a thin gateway with the desire to reduce client round trips?**
Aggregation is fine at the gateway when it's pure composition: "call A and B and return both," with no business decisions. Once it needs logic, move it to a BFF, which exists exactly to reduce round trips for one client. GraphQL is another option for flexible data fetching.

**What parts of this design are general gateway principles, and what parts are specific to Azure APIM?**
General: single entry point, north–south vs east–west, edge auth, rate limits, thin gateway, BFF and domain patterns, versioning, config as code, high availability. APIM-specific: the policy XML, products and subscription keys, versions vs revisions, workspaces, the self-hosted gateway, the tiers and APIOps. Every gateway has equivalents under different names.

**Which links or documentation should attendees use to learn more or try the examples?**
The links on the final slide: the APIM docs, the Azure Architecture Center gateway patterns and microservices guide, microservices.io, Sam Newman's BFF article, Fowler & Lewis's *Microservices* article, and APIOps. To get hands-on, the Microsoft Learn APIM modules and a free Azure account are the easiest start, with a Consumption-tier instance costing very little.

**What assumptions about MyFood's scale, team structure, and workload would change the design?**
- **Scale:** at low traffic, drop Front Door and BFFs and use one gateway. At very high traffic, plan multi-region.
- **Teams:** one team means a single gateway; many teams means domain ownership.
- **Clients:** a single client type doesn't need BFFs.
- **Location:** users only in Lagos means a nearby region and latency testing matter more than global routing.
- **Regulation:** payment and data-protection rules (CBN, NDPA) affect where data lives and what you log.
