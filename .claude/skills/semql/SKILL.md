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
  AND DISTANCE("intermittent drops under load") WITHIN 0.65
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

### 3. Tune cone angles and similarity floors

These numerical parameters control precision vs recall:

| Parameter | Small value | Large value |
|---|---|---|
| `CONE` (radians) | Narrow focus (0.1–0.2), high precision | Broad sweep (0.5–0.8), high recall |
| `WITHIN` (cosine similarity floor, 0–1) | Loose match (0.4–0.6), general vicinity | Tight match (0.8–0.9), near-exact semantics |

`WITHIN` is a **minimum similarity floor**: a candidate passes when its cosine similarity to the anchor is at least the given value. Larger value = stricter neighbourhood. `WITHIN 0` means no floor (every score survives the per-clause gate).

**Starting points for common use cases:**

- Monitoring a specific known pattern: `CONE 0.2`, `WITHIN 0.75`
- Broad topical surveillance: `CONE 0.5`, no `WITHIN`
- Duplicate/near-duplicate detection: `DISTANCE` only, `WITHIN 0.9`
- Concept separation (like X not Y): `CONTRAST` with `WITHIN 0.7`

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
6. **Set numerical parameters** — tune cone angles and similarity floors.
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
- **DISTANCE with too-low `within`** — a floor <0.5 lets almost anything pass, giving a false sense the query is filtered
- **CONTRAST with overlapping attract/repel** — concepts that are semantically close in both lists cancel out
- **Missing NAMESPACE** — queries against the global namespace when they meant org-private
- **NOT applied to the wrong level** — `NOT DISTANCE(x) AND DIRECTION(y)` negates only the distance, not both (operator precedence: NOT binds tighter than AND)
- **Single-clause queries** — almost always too broad; suggest adding a second clause
