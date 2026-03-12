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
PARTITION "org:saas-co"
```

Why it works: DIRECTION alone matches all onboarding content. CONTRAST carves out the failure/friction half of that space. Together they answer "onboarding problems."

### Pattern 2: Specific Signal + Noise Rejection

Find a specific pattern while excluding common false positives. DISTANCE pins to the signal, NOT DISTANCE excludes the noise.

```sql
MATCH DISTANCE("connection pool exhaustion under sustained load") WITHIN 0.3
  AND NOT DISTANCE("connection pool configuration tutorial") WITHIN 0.25
PARTITION "org:platform-team", GLOBAL
```

Why it works: the specific signal and the tutorial content are semantically close (both about connection pools). Without the NOT clause, tutorials would match. The NOT carves out the tutorial neighborhood.

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
PARTITION "org:retailco-*"
```

Why it works: a single broad query would be too noisy. Three targeted OR branches each have high precision in their domain while collectively providing broad coverage.

### Pattern 4: Proximity Refinement

Start broad with DIRECTION, then refine with DISTANCE to focus on a specific manifestation.

```sql
MATCH DIRECTION("infrastructure cost optimization") CONE 0.5
  AND DISTANCE("right-sizing Kubernetes pod resource requests") WITHIN 0.35
PARTITION GLOBAL
```

Why it works: DIRECTION captures the broad topic at high recall. DISTANCE narrows to the specific technique. Messages about cost optimization that aren't about K8s pod sizing are excluded by the DISTANCE threshold, while the DIRECTION ensures we don't drift into unrelated K8s content.

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
      ) WITHIN 0.35
PARTITION "org:platform-team"
```

Why it works: "API rate limiting" is semantically close to many other API concepts. A bare DISTANCE query would match API auth, versioning, etc. The CONTRAST's repel list pushes the composite vector away from the general "API" cluster and toward the specific "rate limiting / throttling" subspace.

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

### Anti-Pattern 3: DISTANCE with large radius (false sense of precision)

```sql
-- BAD: within 0.6 is a huge region — almost anything matches
MATCH DISTANCE("microservice architecture") WITHIN 0.6
```

**Problem:** Cosine distance 0.6 encompasses an enormous volume. This will match anything vaguely related to software architecture. The `WITHIN` gives a false sense that the query is filtered.

**Fix:** Either tighten the radius or switch to DIRECTION:
```sql
-- Option A: tight radius for specific matching
MATCH DISTANCE("microservice architecture") WITHIN 0.2

-- Option B: direction for topical matching + contrast for precision
MATCH DIRECTION("microservice architecture") CONE 0.3
  AND CONTRAST(ATTRACT ["decomposition", "service boundary"], REPEL ["monolith", "modular monolith"])
```

### Anti-Pattern 4: NOT at wrong precedence level

```sql
-- CAUTION: NOT only negates the DISTANCE, not the entire expression
MATCH NOT DISTANCE("routine alert") WITHIN 0.2
  AND DIRECTION("infrastructure") CONE 0.4
```

This is equivalent to `(NOT DISTANCE(...)) AND DIRECTION(...)` — it matches everything about infrastructure that ISN'T a routine alert. This might be what you want, but make sure. If you want to negate the whole thing:

```sql
-- Negate the entire group:
MATCH NOT (DISTANCE("routine alert") WITHIN 0.2 AND DIRECTION("infrastructure") CONE 0.4)
```

### Anti-Pattern 5: Using partition as a semantic filter

```sql
-- BAD: partitions are access boundaries, not topic labels
MATCH DISTANCE("bug report")
PARTITION "topic:frontend-bugs"
```

**Problem:** Partitions are organizational isolation boundaries (org, team, session), not semantic categories. There is no `topic:` partition scheme.

**Fix:** Use clauses for semantic narrowing:
```sql
MATCH DISTANCE("bug report") WITHIN 0.3
  AND DIRECTION("frontend rendering") CONE 0.3
PARTITION "org:acme-corp"
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
  AND NOT DISTANCE("routine security scan results") WITHIN 0.2
PARTITION "org:{{org}}", GLOBAL
```

### Customer Churn Signals

```sql
MATCH DIRECTION(["customer dissatisfaction", "cancellation intent"]) CONE 0.4
  AND CONTRAST(
        ATTRACT ["enterprise", "high-value", "long-term customer"],
        REPEL   ["trial user", "free tier", "just browsing"]
      )
PARTITION "org:{{org}}"
WINDOW 7d
```

### Technical Debt Detection

```sql
MATCH DIRECTION("technical debt") CONE 0.4
  AND CONTRAST(
        ATTRACT ["workaround", "hack", "TODO", "known issue", "legacy"],
        REPEL   ["refactored", "cleaned up", "resolved", "migrated"]
      )
PARTITION "org:{{org}}"
```

### Competitive Intelligence

```sql
MATCH DIRECTION(["competitor product launch", "market disruption"]) CONE 0.4
  AND CONTRAST(
        ATTRACT ["{{competitor_name}}", "market share", "pricing change"],
        REPEL   ["{{own_company}}", "internal roadmap", "our product"]
      )
PARTITION GLOBAL
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
PARTITION "org:{{org}}"
WINDOW 1h
```
