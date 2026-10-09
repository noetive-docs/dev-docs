# SemQL Patterns and Anti-Patterns

## Effective Patterns

### Pattern 1: Topic + Polarity

Monitor a topic with a specific sentiment or outcome direction. DIRECTION defines the topic, CONTRAST separates positive from negative.

```sql
MATCH DIRECTION("customer onboarding") CONE 0.4
  AND CONTRAST(
        ATTRACT ["friction", "confusion", "drop-off", "abandoned"],
        REPEL   ["success", "completed", "activated", "converted"]
      )
NAMESPACE "dinoco"
```

Why it works: DIRECTION alone matches all onboarding content. CONTRAST carves out the failure/friction half of that space. Together they answer "onboarding problems."

### Pattern 2: Specific Signal + Noise Rejection

Find a specific pattern while excluding common false positives. DISTANCE pins to the signal, NOT DISTANCE excludes the noise.

```sql
MATCH DISTANCE("connection pool exhaustion under sustained load")
  AND NOT DISTANCE("connection pool configuration tutorial")
NAMESPACE "gusteau", GLOBAL
```

Why it works: the specific signal and the tutorial content are semantically close (both about connection pools). The first DISTANCE keeps only messages close to the signal. The NOT DISTANCE excludes messages closer to the tutorial anchor than general text is, carving out the tutorial neighbourhood. Inside the AND the NOT only filters; it does not raise the score.

### Pattern 3: Multi-Domain Surveillance

Watch for the same class of problem across different domains. OR branches cover each domain, AND within each branch provides precision.

```sql
MATCH (
        DIRECTION("payment processing") CONE 0.3
    AND CONTRAST(ATTRACT ["timeout", "failure"], REPEL ["success", "completed"])
  )
  OR (
        DIRECTION("inventory management") CONE 0.3
    AND CONTRAST(ATTRACT ["stockout", "discrepancy"], REPEL ["balanced", "reconciled"])
  )
  OR (
        DIRECTION("shipping fulfillment") CONE 0.3
    AND CONTRAST(ATTRACT ["delay", "lost package"], REPEL ["delivered", "on time"])
  )
NAMESPACE "axiom"
```

Why it works: a single broad query would be too noisy. Three targeted OR branches each have high precision in their domain while collectively providing broad coverage.

### Pattern 4: Proximity Refinement

Start broad with DIRECTION, then refine with DISTANCE to focus on a specific manifestation.

```sql
MATCH DIRECTION("infrastructure cost optimization") CONE 0.5
  AND DISTANCE("right-sizing Kubernetes pod resource requests")
NAMESPACE GLOBAL
```

Why it works: DIRECTION captures the broad topic at high recall. DISTANCE narrows to the specific technique. Messages about cost optimization that are not close to the K8s pod-sizing anchor are dropped, while the DIRECTION ensures we don't drift into unrelated K8s content.

### Pattern 5: Concept Boundary

Define a concept by what it IS and what it ISN'T, using CONTRAST with detailed attract/repel lists.

```sql
MATCH CONTRAST(
        ATTRACT [
          "API rate limiting",
          "request throttling",
          "backpressure mechanism",
          "load shedding"
        ],
        REPEL [
          "API authentication",
          "API versioning",
          "API documentation",
          "API design patterns"
        ]
      )
NAMESPACE "gusteau"
```

Why it works: "API rate limiting" is semantically close to many other API concepts. A bare DISTANCE query would match API auth, versioning, etc. The CONTRAST's repel list pushes the composite vector away from the general "API" cluster and toward the specific "rate limiting / throttling" subspace.

### Pattern 6: Shared work with groups

Subscribe several roles with the same `group` so each message goes to its highest-scoring member only. Members must match on the same clause kinds (DISTANCE, DIRECTION, CONTRAST), or the subscription is refused with `400 invalid_request`. For DISTANCE the members must also use the same metric (mixing metrics is refused like mixing kinds). Write members in the same shape as well: an AND member scores the mean of its clauses and an OR member the best, so an AND member loses to an OR member it would otherwise tie. Groups belong to the account, so all your keys share its group names. A member's seq has gaps for messages a peer won, so a gap does not by itself mean a lost message; messages are still dropped when the winning member's stream is not read fast enough. Only open streams compete: once Semantik sees a member's stream has closed, the next-closest open member receives what it would have won, and until then messages that member wins are lost with the stream. Give each role a few short anchors in its own vocabulary plus one OR'd sentence naming what it is responsible for. Known limit: an on-topic message that is nobody's job still reaches the closest role, so consumers should be able to decline.

```sql
MATCH DISTANCE("deploy failed") OR DISTANCE("rollback")
  OR DISTANCE("This role owns release engineering: deployments, rollbacks and CI pipeline failures")
```

---

## Anti-Patterns

### Anti-Pattern 1: Single-clause DIRECTION (too broad)

```sql
-- BAD: matches everything even remotely about payments
MATCH DIRECTION("payments") CONE 0.5
```

**Problem:** DIRECTION with a wide cone and a generic concept matches enormous volumes. "Payments" is a vast semantic region.

**Fix:** Add CONTRAST to narrow, or use a more specific anchor, or tighten the cone:
```sql
MATCH DIRECTION("payment gateway integration") CONE 0.3
  AND CONTRAST(ATTRACT ["error", "failure"], REPEL ["documentation", "tutorial"])
```

### Anti-Pattern 2: Overlapping attract/repel in CONTRAST

```sql
-- BAD: "performance" and "optimization" are semantically close
MATCH CONTRAST(
        ATTRACT ["performance optimization", "speed improvement"],
        REPEL   ["performance testing", "performance monitoring"]
      )
```

**Problem:** The attract and repel concepts share the word "performance" and are semantically adjacent. The repel vectors partially cancel the attract vectors, producing a weak, unstable composite direction.

**Fix:** Make attract and repel semantically distant:
```sql
MATCH DIRECTION("performance optimization") CONE 0.3
  AND CONTRAST(
        ATTRACT ["faster", "reduced latency", "throughput improvement"],
        REPEL   ["testing framework", "monitoring dashboard", "alerting setup"]
      )
```

### Anti-Pattern 3: Setting WITHIN (refused)

```sql
-- BAD: 400 invalid_request
MATCH DISTANCE("microservice architecture") WITHIN 0.8
```

**Problem:** SemQL has a `WITHIN` modifier but Semantik refuses it on every clause; /v1/lint reports it as an error. Semantik sets the relevance bar itself, per clause: a clause holds only when the message is closer to it than general reference text is. `A AND B` needs each clause to hold on its own, and clauses of one kind ORed together share one bar that rises with the group's breadth.

**Fix:** To match less, make the anchor text more specific or add a CONTRAST; to match more, `OR` in another anchor of similar specificity: anchors of one kind ORed together share one bar, so ORing in a much broader anchor raises the bar for the narrower ones and can drop their marginal matches.
```sql
MATCH DISTANCE("microservice decomposition and service boundaries")

MATCH DIRECTION("microservice architecture") CONE 0.9
  AND CONTRAST(ATTRACT ["decomposition", "service boundary"], REPEL ["monolith", "modular monolith"])
```

### Anti-Pattern 4: NOT at wrong precedence level

```sql
-- CAUTION: NOT only negates the DISTANCE, not the entire expression (valid: NOT sits directly under an AND with a positive clause)
MATCH NOT DISTANCE("routine alert")
  AND DIRECTION("infrastructure") CONE 0.4
```

This is equivalent to `(NOT DISTANCE(...)) AND DIRECTION(...)` — it matches everything about infrastructure that ISN'T close enough to a routine alert. This might be what you want, but make sure. To exclude several things at once, negate a group, still directly under the `AND` with its positive clause. A top-level `NOT (A AND B)`, or `NOT` under an `OR`, is refused with `400 invalid_request` because it would match nearly every message:

```sql
MATCH DIRECTION("infrastructure") CONE 0.4
  AND NOT (DISTANCE("routine alert") OR DISTANCE("scheduled maintenance"))
```

### Anti-Pattern 5: Using namespace as a semantic filter

```sql
-- BAD: namespaces are access boundaries, not topic labels
MATCH DISTANCE("bug report")
NAMESPACE "topic:frontend-bugs"
```

**Problem:** Namespaces are organizational isolation boundaries (org, team, session), not semantic categories. There is no `topic:` namespace scheme.

**Fix:** Use clauses for semantic narrowing:
```sql
MATCH DISTANCE("bug report")
  AND DIRECTION("frontend rendering") CONE 0.3
NAMESPACE "monsters"
```

---

## Domain Templates

### Security Monitoring

```sql
MATCH DIRECTION(["security vulnerability", "exploit", "unauthorized access"]) CONE 0.35
  AND CONTRAST(
        ATTRACT ["production", "critical", "customer-facing"],
        REPEL   ["test environment", "CTF", "educational", "training exercise"]
      )
  AND NOT DISTANCE("routine security scan results")
NAMESPACE "{{org}}", GLOBAL
```

### Customer Churn Signals

```sql
MATCH DIRECTION(["customer dissatisfaction", "cancellation intent"]) CONE 0.4
  AND CONTRAST(
        ATTRACT ["enterprise", "high-value", "long-term customer"],
        REPEL   ["trial user", "free tier", "just browsing"]
      )
NAMESPACE "{{org}}"
WINDOW 7d
```

### Technical Debt Detection

```sql
MATCH DIRECTION("technical debt") CONE 0.4
  AND CONTRAST(
        ATTRACT ["workaround", "hack", "TODO", "known issue", "legacy"],
        REPEL   ["refactored", "cleaned up", "resolved", "migrated"]
      )
NAMESPACE "{{org}}"
```

### Competitive Intelligence

```sql
MATCH DIRECTION(["competitor product launch", "market disruption"]) CONE 0.4
  AND CONTRAST(
        ATTRACT ["{{competitor_name}}", "market share", "pricing change"],
        REPEL   ["{{own_company}}", "internal roadmap", "our product"]
      )
NAMESPACE GLOBAL
```

### Infrastructure Incident Correlation

```sql
MATCH (
        DIRECTION("database performance degradation") CONE 0.3
    AND CONTRAST(ATTRACT ["spike", "latency", "timeout"], REPEL ["planned", "maintenance"])
  )
  OR (
        DIRECTION("service mesh failure") CONE 0.3
    AND CONTRAST(ATTRACT ["cascade", "circuit breaker", "retry storm"], REPEL ["deployment", "rollout"])
  )
NAMESPACE "{{org}}"
WINDOW 1h
```
