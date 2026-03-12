# SemQL v1.0 Reference

## Grammar (EBNF)

```ebnf
query       = "MATCH" expression [ "PARTITION" partition ] [ "WINDOW" duration ] [ "LIMIT" integer ]
expression  = term { "OR" term }
term        = factor { "AND" factor }
factor      = [ "NOT" ] ( clause | "(" expression ")" )
clause      = distance | direction | contrast

distance    = "DISTANCE" "(" anchor ")" [ "WITHIN" number | "TOP" integer ]
direction   = "DIRECTION" "(" anchor_list ")" [ "CONE" number ]
contrast    = "CONTRAST" "(" "ATTRACT" anchor_list "," "REPEL" anchor_list ")" [ "WITHIN" number ]

partition   = partition_ref { "," partition_ref } | "ALL" | "GLOBAL"
partition_ref = [ "NOT" ] string

anchor      = string | vector
anchor_list = "[" anchor { "," anchor } "]" | anchor
vector      = "[" number { "," number } "]"
duration    = integer ( "s" | "m" | "h" | "d" | "w" )
```

Operator precedence: NOT (tightest) > AND > OR (loosest).

All reserved words are case-insensitive: `MATCH AND OR NOT DISTANCE DIRECTION CONTRAST ATTRACT REPEL WITHIN TOP CONE WINDOW PARTITION ALL GLOBAL LIMIT`.

## Clauses

### DISTANCE — sphere in embedding space

| Field | Type | Required | Default | Range |
|---|---|---|---|---|
| `anchor` | string or float[] | yes | — | — |
| `within` | number | no | — | 0.0–2.0 (cosine distance) |
| `top_k` | integer | no | — | ≥1 |
| `metric` | string | no | `"cosine"` | `"cosine"`, `"euclidean"`, `"dot"` |

Cannot specify both `within` and `top_k`. If neither is set, the clause scores without thresholding.

Text: `DISTANCE("payment failure") WITHIN 0.3`
JSON: `{"distance": {"anchor": "payment failure", "within": 0.3}}`

### DIRECTION — cone in embedding space

| Field | Type | Required | Default | Range |
|---|---|---|---|---|
| `toward` | string, string[], or float[] | yes | — | — |
| `cone` | number | no | 0.3 | 0.0 (exact) to π/2 (hemisphere) |

When `toward` is an array, the direction vector is the normalized mean of all embedded concepts.

Text: `DIRECTION(["customer frustration", "billing"]) CONE 0.4`
JSON: `{"direction": {"toward": ["customer frustration", "billing"], "cone": 0.4}}`

### CONTRAST — attract/repel vector arithmetic

| Field | Type | Required | Default | Range |
|---|---|---|---|---|
| `attract` | string[] | yes | — | ≥1 item |
| `repel` | string[] | yes | — | ≥1 item |
| `within` | number | no | — | 0.0–2.0 |

Composite vector: `normalize(mean(embed(attract)) - mean(embed(repel)))`.

Text: `CONTRAST(ATTRACT ["enterprise"], REPEL ["free tier"]) WITHIN 0.4`
JSON: `{"contrast": {"attract": ["enterprise"], "repel": ["free tier"], "within": 0.4}}`

## Expressions (JSON)

```json
{"and": [expr, expr, ...]}
{"or":  [expr, expr, ...]}
{"not": expr}
{"distance": {...}}          // bare clause = single expression
```

`and` and `or` require ≥2 items. `not` takes exactly one expression.

## Partition Selector (JSON)

```json
{
  "include": ["org:acme-corp", "org:acme-*"],
  "exclude": ["org:acme-staging"],
  "global": true
}
```

Supports glob patterns (`*`, `?`). `global: true` includes the global partition.

## Duration

Text shorthand: `30s`, `15m`, `24h`, `7d`, `4w`.
JSON: ISO 8601 — `"PT30S"`, `"PT15M"`, `"PT24H"`, `"P7D"`, `"P28D"`.

## Full Query (JSON)

```json
{
  "match": <expression>,
  "partition": <partition_selector>,
  "window": "<ISO 8601 duration>",
  "limit": <integer>
}
```

Only `match` is required.

## Future work (NOT in v1.0)

Trajectory, Region, Anomaly, Resonance clauses. Named references (`$var`, `MSG()`, `CENTROID()`). Composite anchors. Clause weights.
