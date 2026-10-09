---
name: semql
description: >
  Write, review, and debug SemQL (Semantic Query Language) queries for the Noetive broker.
  Use this skill whenever the user mentions SemQL, Noetive, semantic subscriptions, semantic queries,
  embedding space queries, vector matching, or needs to express interest in a region of semantic space.
  Also trigger when the user wants to create a subscription that monitors for semantic patterns,
  asks about writing queries that combine distance/direction/contrast clauses,
  asks how to express "like X but not Y" in a query language, or mentions matching against
  high-dimensional embeddings. Even if the user says something general like "I want to watch for
  messages about X", consider whether SemQL is the right tool and trigger this skill.
  This skill covers both the SQL-like text syntax and the JSON wire format.
---

# SemQL Writing Skill

Write precise, effective SemQL queries for the Noetive semantic broker.

## When to use this skill

- User wants to create a SemQL query (text or JSON)
- User wants to create or modify a Noetive subscription
- User needs to express semantic interest in a region of embedding space
- User wants to debug or optimize an existing SemQL query
- User asks "how do I match things about X but not Y" or similar
- User mentions DISTANCE, DIRECTION, CONTRAST clauses

## Before writing any SemQL

Read the reference spec first to ensure correctness:

```
references/semql-spec-v1.md    — Full language specification (grammar, clauses, JSON schema)
references/patterns.md         — Common query patterns and anti-patterns
```

**Always read `references/semql-spec-v1.md` before writing or reviewing SemQL.** The spec is the source of truth for syntax, clause semantics, and valid field values.

## Core principles

### 1. Choose the right clause for the geometric intent

Each clause answers a different question about the embedding space. Selecting the wrong clause produces a query that appears to work but matches the wrong content.

| User intent | Clause | Geometry |
|---|---|---|
| "Find messages similar to X" | `DISTANCE` | Sphere around a point |
| "Find messages about the topic of X" | `DIRECTION` | Cone aligned with a direction |
| "Find messages like X but not Y" | `CONTRAST` | Attract/repel vector arithmetic |
| "Find messages similar to X about the topic of Y" | `DISTANCE` + `DIRECTION` | Sphere intersected with cone |

**DISTANCE** cares about proximity — both topic AND specificity matter. A brief mention and a deep analysis of the same topic are at different distances from the anchor.

**DIRECTION** cares about thematic alignment only — it ignores magnitude. A brief mention and a deep analysis point in the same direction and both match. Use `DIRECTION` when you want topical coverage regardless of depth.

**CONTRAST** creates a composite vector from attract/repel concepts. It answers "in the direction of A, away from B." Use it to carve out a specific slice of the space that can't be expressed with a single anchor point.

### 2. Combine clauses to sculpt precise regions

Single-clause queries are almost always too broad. The power of SemQL is clause composition.

**Good pattern — narrowing with AND:**
```sql
MATCH DIRECTION(["payment processing", "transaction"]) CONE 0.4
  AND CONTRAST(
        ATTRACT ["failure", "error", "timeout"],
        REPEL   ["success", "completed", "processed"]
      )
  AND DISTANCE("intermittent drops under load")
```

This says: thematically about payments (DIRECTION), specifically the failure/error side (CONTRAST), and close to the specific pattern of intermittent drops (DISTANCE). Each clause eliminates a different kind of noise.

**Good pattern — broadening with OR:**
```sql
MATCH (
        DIRECTION("database connection pooling") CONE 0.3
    AND CONTRAST(ATTRACT ["exhaustion", "leak"], REPEL ["normal", "healthy"])
  )
  OR (
        DIRECTION("HTTP client timeout") CONE 0.3
    AND CONTRAST(ATTRACT ["cascade", "retry storm"], REPEL ["single request", "isolated"])
  )
```

This watches for two distinct failure modes that might have the same downstream impact.

### 3. Tune cone angles, not thresholds

**There is no similarity threshold to tune.** Semantik sets the relevance bar itself, per clause: a clause holds for a message only when the message is closer to it than general reference text is. `A AND B` matches only when each clause holds on its own; a strong match on one does not make up for the other. Clauses of one kind ORed together (`A OR B OR C`, all DISTANCE, say) share one bar measured on the group as a whole, which rises with the group's breadth. Clauses of different kinds in one OR (DISTANCE, DIRECTION, CONTRAST) each keep their own bar, so a CONTRAST branch still fires next to DISTANCE branches. To match less, write more specific anchor text; to match more, `OR` in another anchor of similar specificity: anchors of one kind ORed together share one bar, so ORing in a much broader anchor raises the bar for the narrower ones and can drop their marginal matches. `CONE` can only tighten: a message must be inside the cone and above the bar, so a cone wider than the bar changes nothing.

SemQL has a `WITHIN` modifier, but Semantik refuses it: a query that sets `WITHIN` on any clause (DISTANCE or CONTRAST, text or JSON `within`) gets `400 invalid_request` from `/v1/search` and `/v1/subscribe`, and `/v1/lint` reports it as an error. Never use it in docs examples.

The one numeric knob is `CONE` on `DIRECTION` (radians; lower is tighter).

**Starting points for common use cases:**

- Monitoring a specific known pattern: `CONE 0.2` plus a specific `DISTANCE` anchor
- Broad topical surveillance: `CONE 0.5`
- Near-duplicate detection: `DISTANCE` with a specific anchor
- Concept separation (like X not Y): `CONTRAST`

### 3b. NOT, and groups for one owner per message

`NOT` is supported only directly under an `AND` that also has a positive clause. `A AND NOT DISTANCE("x")` excludes messages closer to x than general reference text is; the `NOT` only filters and does not raise the score. Anywhere else (`A OR NOT B`, `NOT A AND NOT B`, `A AND (B OR NOT C)`) the query is refused with `400 invalid_request`, because it would match nearly every message.

Several subscriptions that subscribe with the same `group` name (1–64 of letters, digits, `-`, `_`) share a stream: a message matching several members goes only to the highest-scoring one, and grouped match frames carry `lead`, the margin over the best other matching member. Members must match on the same clause kinds (DISTANCE, DIRECTION, CONTRAST; clauses under `NOT` ignored); a subscription whose kinds differ from the group's current members is refused with `400 invalid_request`. For DISTANCE the members must also use the same metric (mixing metrics is refused like mixing kinds). Write members in the same shape as well: an AND member scores the mean of its clauses and an OR member the best, so an AND member loses to an OR member it would otherwise tie. Groups belong to the account, so every API key on the account shares its group names and another account's identically named group never competes with yours. For a role, use a few short anchors in the words its messages use, plus one OR'd sentence stating what the role is responsible for.

### 4. Use NAMESPACE to control scope, not semantics

Namespaces are isolation boundaries, not semantic categories. Don't use namespaces to narrow meaning — use clauses for that.

**Wrong — using namespace as a topic filter:**
```sql
MATCH DISTANCE("connection timeout")
NAMESPACE "topic:networking"       -- This is not how namespaces work
```

**Right — using namespace as an access boundary:**
```sql
MATCH DIRECTION("connection timeout") CONE 0.3
  AND CONTRAST(ATTRACT ["production"], REPEL ["test", "staging"])
NAMESPACE "monsters", GLOBAL
```

### 5. Prefer text anchors over raw vectors

Text anchors are readable, maintainable, and the broker embeds them with the same model used for messages. Raw vectors are opaque and brittle across model versions.

**Prefer:**
```sql
MATCH DISTANCE("payment gateway returning 202 async responses")
```

**Avoid (unless you have a specific reason):**
```sql
MATCH DISTANCE([0.182, -0.041, 0.389, 0.057, ...])
```

Raw vectors are appropriate only when you already have a pre-computed embedding (e.g., from a message you want to find neighbors of).

### 6. Write the JSON format for machines, the text format for humans

Both formats are equivalent. Use text syntax in documentation, discussions, and human-facing contexts. Use JSON in API calls, config files, and code.

When outputting SemQL, always provide **both formats** unless the user specifies one.

## Query construction process

When a user describes what they want to monitor, follow this process:

1. **Identify the core concept** — what topic or region of meaning are they interested in?
2. **Identify exclusions** — what should NOT match? This becomes a CONTRAST repel or a NOT clause.
3. **Identify scope** — which namespaces, what time window, what volume limit?
4. **Choose clause types** — map each aspect to the right geometric primitive.
5. **Compose with AND/OR/NOT** — combine clauses to sculpt the region.
6. **Set numerical parameters** — tune cone angles; there is no similarity floor to set.
7. **Write both formats** — provide text and JSON.
8. **Explain the query** — describe what each clause does and why it's there.

## Output format

When writing SemQL, always structure your response as:

1. **The query in text syntax** (in a `sql` code block)
2. **The query in JSON format** (in a `json` code block)
3. **A brief explanation** of each clause's purpose and how they interact
4. **Tuning notes** — which parameters the user might want to adjust and what the tradeoffs are

## Common mistakes to catch when reviewing SemQL

Read `references/patterns.md` for the full catalog. The most frequent mistakes:

- **DIRECTION without CONTRAST** — too broad, matches anything topically adjacent
- **Setting `WITHIN`** — Semantik refuses it with `400 invalid_request`; make the anchor more specific, or `OR` in another anchor of similar specificity, instead
- **CONTRAST with overlapping attract/repel** — concepts that are semantically close in both lists cancel out
- **Missing NAMESPACE** — queries against the global namespace when they meant org-private
- **NOT applied to the wrong level** — `NOT DISTANCE(x) AND DIRECTION(y)` negates only the distance, not both (operator precedence: NOT binds tighter than AND)
- **NOT anywhere but directly under an AND with a positive clause** — `A OR NOT B`, `NOT A AND NOT B` and `A AND (B OR NOT C)` are refused with `400 invalid_request`
- **Single-clause queries** — almost always too broad; suggest adding a second clause
