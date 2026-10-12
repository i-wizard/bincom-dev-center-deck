# Part 3 — Core Responsibilities (~5 min)

## Slide 6 — Routing & Transformation

**On slide:**

**Routing:** send each request to the right service
- By path: `/orders/*` → Orders service, `/menu/*` → Restaurants & Menu
- By header: send 10% of traffic to a new version (canary release)

**Transformation:** reshape requests and responses
- Rewrite URLs, add/remove headers, reshape payloads

**Aggregation:** combine several calls into one response
- MyFood home screen = restaurants + promos + active order, in one call

**APIM policy example:**

```xml
<inbound>
    <base />
    <set-backend-service backend-id="orders-service" />
    <rewrite-uri template="/api/orders/{orderId}" />
</inbound>
```

**Visual:** Left: a small diagram of `GET /orders/123` entering the gateway and being routed to the Orders service, with the path rewritten. Right: **screenshots** of (1) the APIM APIs list showing the MyFood APIs and (2) the policy editor with the policy above.

**Speaker notes (~1.5 min):**
- Routing: "The most basic job. The gateway looks at the path or a header and decides which service gets the request. Clients only know `api.myfood.ng/orders`; they never learn where the Orders service actually lives. That means we can move or split services without breaking clients."
- Canary: "Because the gateway controls routing, it can also send a small percentage of traffic to a new version, so we can test it safely in production."
- Transformation: "The gateway can rewrite URLs, add headers, or reshape payloads. Useful when the public API shape differs from the internal one."
- Aggregation: "When the customer opens MyFood, the home screen needs restaurants, promos and their active order. Instead of three round trips from the phone, the gateway can make those calls and return one response."
- APIM: "In APIM, all of this is done with *policies*. A policy is a small piece of XML configuration that runs on every request. Here we pick the backend and rewrite the URL."
- Caution: "APIM *can* do aggregation with the `send-request` policy, but it gets messy quickly. For anything beyond simple composition, put it in a small backend-for-frontend service. More on that in Part 4."
- Naming: "Microsoft documents these as the *Gateway Routing* and *Gateway Aggregation* patterns, if you want to read further."

---

## Slide 7 — Security at the Edge

**On slide:**

Check identity **once**, at the gate:
- 🔒 **TLS termination:** HTTPS ends at the gateway
- 🪪 **Authentication:** validate the JWT access token on every request
- 🔑 **Subscription keys:** identify partner integrations
- ➡️ **Pass identity on:** forward the user ID to downstream services

> The gateway checks **who you are**. The service still decides **what you're allowed to do**.

**APIM policy example:**

```xml
<validate-jwt header-name="Authorization"
              failed-validation-httpcode="401"
              failed-validation-error-message="Unauthorized">
    <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
    <audiences>
        <audience>api://myfood-api</audience>
    </audiences>
</validate-jwt>
```

**Visual:** A request with a token arriving at the gateway. A "✅ valid" path continues to the Orders service with an `X-User-Id` header added; a "❌ invalid" path returns `401` straight from the gateway. Right: **screenshots** of (1) the policy editor with `validate-jwt` and (2) **Products & subscriptions** showing the partner subscription keys.

**Speaker notes (~1.5 min):**
- "This is the estate security guard. Instead of every service checking tokens, the gateway does it once."
- TLS: "HTTPS ends at the gateway, so certificates are managed in one place."
- Authentication: "The customer signs in with Microsoft Entra ID (Entra External ID for customer accounts) and gets a token. The gateway validates that token on every request. Bad or expired tokens get a 401 before they ever reach our services."
- Partners: "Partners like restaurant POS systems get a subscription key. In APIM, you group APIs into *products*, and partners subscribe to a product to get a key. That also tells us exactly which partner is calling."
- Passing identity: "After validating the token, the gateway can forward the user ID as a header, so services don't need to parse tokens again."
- Important nuance: "The gateway handles *authentication*: who are you? The service still handles *authorisation*: can *this* customer cancel *this* order? That's a business rule, so it stays in the service."
- "Also lock the network down so services can *only* be reached through the gateway. Otherwise someone can just walk around the gate."
- Naming: "Microsoft calls this the *Gateway Offloading* pattern: moving shared concerns off the services and onto the gateway."

---

## Slide 8 — Traffic Control & Observability

**On slide:**

**Traffic control:** protect your services
- **Rate limits:** e.g. 100 calls/minute per partner → `429 Too Many Requests`
- **Quotas:** e.g. 10,000 calls/day per partner
- **Timeouts:** don't let a slow service hold requests forever

**Observability:** see everything in one place
- A **correlation ID** on every request, passed to every service
- Central logs, metrics and traces

**APIM policy example:**

```xml
<rate-limit-by-key calls="100" renewal-period="60"
                   counter-key="@(context.Subscription.Id)" />
<quota-by-key calls="10000" renewal-period="86400"
              counter-key="@(context.Subscription.Id)" />
<set-header name="X-Correlation-Id" exists-action="skip">
    <value>@(context.RequestId.ToString())</value>
</set-header>
```

**Visual:** Left: a partner sending lots of requests; the first 100 pass and the rest get `429`. Below it, one request with a correlation ID flowing through the gateway → Orders → Payments, with the same ID appearing in each log line. Right: **screenshots** of (1) the policy editor with the policy above and (2) **APIM Analytics or Application Insights** showing request counts and response times.

**Speaker notes (~2 min):**
- Rate limiting: "Imagine a partner's POS system has a bug and sends thousands of requests a second. Without a limit, it can take down our Orders service for *every* customer. With a limit per subscription key, only that partner gets `429 Too Many Requests`."
- Quotas: "Quotas are the long-term version: how many calls a partner can make per day or month. Useful for pricing tiers too."
- Timeouts: "If a service is slow, the gateway should give up after a set time instead of holding the connection forever. I'm keeping this light; resilience is a whole talk on its own."
- Observability: "Because every request passes through the gateway, it's the perfect place to start tracing. The gateway adds a correlation ID, every service logs it, and when a customer says 'my order failed', you can search one ID and see the whole journey."
- APIM: "APIM connects to Application Insights for logs, metrics and distributed tracing, and has a built-in Analytics view showing traffic per API, per operation and per subscriber."
- Analogy: "This is the estate gate's logbook, plus a guard who stops one visitor from bringing in 200 friends."
