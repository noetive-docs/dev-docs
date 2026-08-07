# SemQL v1.0 Reference

## Grammar (EBNF)

```ebnf
query       = "MATCH" expression [ "NAMESPACE" namespace ] [ "WINDOW" duration ] [ "LIMIT" integer ]
expression  = term { "OR" term }
term        = factor { "AND" factor }
factor      = [ "NOT" ] ( clause | "(" expression ")" )
clause      = distance | direction | contrast

distance    = "DISTANCE" "(" anchor ")" [ "WITHIN" number | "TOP" integer ]
direction   = "DIRECTION" "(" anchor_list ")" [ "CONE" number ]
contrast    = "CONTRAST" "(" "ATTRACT" anchor_list "," "REPEL" anchor_list ")" [ "WITHIN" number ]

namespace      = namespace_item { "," namespace_item }
namespace_item = namespace_ref | "ALL" | "GLOBAL"
namespace_ref  = [ "NOT" ] string

anchor      = string | vector
anchor_list = "[" anchor { "," anchor } "]" | anchor
vector      = "[" number { "," number } "]"
duration    = integer ( "s" | "m" | "h" | "d" | "w" )
```

Operator precedence: NOT (tightest) > AND > OR (loosest).

All reserved words are case-insensitive: `MATCH AND OR NOT DISTANCE DIRECTION CONTRAST ATTRACT REPEL WITHIN TOP CONE WINDOW NAMESPACE ALL GLOBAL LIMIT`.

## Clauses

### DISTANCE — sphere in embedding space

| Field | Type | Required | Default | Range |
|---|---|---|---|---|
| `anchor` | string or float[] | yes | — | — |
| `within` | number | no | — | 0.0–1.0 (cosine similarity floor; 0 = no floor) |
| `top_k` | integer | no | — | ≥1 |
| `metric` | string | no | `"cosine"` | `"cosine"`, `"euclidean"`, `"dot"` |

`within` is a minimum similarity floor — a match passes when its cosine similarity to the anchor is at least `within`. Larger value = stricter neighbourhood.

Cannot specify both `within` and `top_k`. If neither is set, the clause scores without thresholding.

Text: `DISTANCE("payment failure") WITHIN 0.7`
JSON: `{"distance": {"anchor": "payment failure", "within": 0.7}}`

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
| `repel` | string[] | no | — | ≥1 item |
| `within` | number | no | — | 0.0–1.0 (cosine similarity floor from composite vector; 0 = no floor) |

Composite vector: `normalize(mean(embed(attract)) - mean(embed(repel)))` when `repel` is present; `normalize(mean(embed(attract)))` otherwise.

Text: `CONTRAST(ATTRACT ["enterprise"], REPEL ["free tier"]) WITHIN 0.6`
JSON: `{"contrast": {"attract": ["enterprise"], "repel": ["free tier"], "within": 0.6}}`

## Expressions (JSON)

```json
{"and": [expr, expr, ...]}
{"or":  [expr, expr, ...]}
{"not": expr}
{"distance": {...}}          // bare clause = single expression
```

`and` and `or` require ≥2 items. `not` takes exactly one expression.

## Namespace Selector (JSON)

```json
{
  "include": ["monsters"],
  "exclude": ["monsters-staging"],
  "global": true,
  "all": false
}
```

Namespace names are matched literally — no wildcards. `global: true` includes the global namespace; `all: true` includes every namespace the caller has access to. The text-form keywords `GLOBAL` and `ALL` map to those booleans and may appear alongside literal names in the comma-separated list (e.g. `NAMESPACE "monsters", GLOBAL`).

## Duration

Text shorthand: `30s`, `15m`, `24h`, `7d`, `4w`.
JSON: ISO 8601 — `"PT30S"`, `"PT15M"`, `"PT24H"`, `"P7D"`, `"P28D"`.

## Full Query (JSON)

```json
{
  "match": <expression>,
  "namespace": <namespace_selector>,
  "window": "<ISO 8601 duration>",
  "limit": <integer>
}
```

Only `match` is required.
