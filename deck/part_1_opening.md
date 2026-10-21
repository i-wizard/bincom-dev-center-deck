# Part 1 — Opening (~2 min)

## Slide 1 — Title

**On slide:**

> # Designing the Gateway Layer in a Microservices Architecture
> David Njoagwuani · Senior Software Engineer, CentricAI
> Bincom Dev Center Virtual Meetup · [Date]

**Visual:** Clean title slide. A small simple icon on the right: three client icons (phone, laptop, partner) → one gate → several service boxes. This previews the core idea of the talk.

**Speaker notes (~30s):**
- Greet everyone and thank Bincom Dev Center for hosting.
- "Today we're talking about the front door of a microservices system: the gateway layer. What it does, how to design it, and the mistakes that make it painful."
- "We'll use one example the whole way through, a food delivery app called MyFood, and look at how it's built on Azure API Management."

---

## Slide 2 — About Me + Agenda

**On slide:**

**About me**
- Senior Software Engineer @ CentricAI
- 7+ years building backend systems & APIs (Python, Django, FastAPI)
- Microservices · AWS · Azure · Kubernetes · Messaging · CI/CD
- Currently: AI & data-intensive platforms
- Interested in backend architecture & distributed systems at scale

**Agenda**
1. Why a gateway layer?
2. What a gateway does
3. Gateway patterns
4. Key design decisions
5. Pitfalls to avoid
6. Putting it together: MyFood on Azure

**Visual:** Two-column layout. Left: photo + bio bullets. Right: numbered agenda. Optionally, small icons next to each agenda item.

**Speaker notes (~1.5 min):**
- Quick intro: "I'm David, a backend engineer at CentricAI. I've spent most of the last seven years building APIs and backend systems, many of them as microservices on AWS and Azure."
- Why this topic: "In almost every microservices system I've worked on, the gateway was either the thing that made life easy or the thing everyone complained about. The difference was almost always design, not the tool."
- Walk the agenda: "We start with *why* you need a gateway, then *what* it should do, then the main *patterns* for structuring it, the *decisions* you'll have to make, the *pitfalls*, and finally we put it all together with MyFood on Azure."
- Set expectations:  you'll see diagrams and real screenshots from Azure API Management. Hold questions to the end; we'll have about five minutes."
