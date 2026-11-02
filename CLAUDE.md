# CLAUDE.md

Developer documentation site built with Mintlify. When writing or reviewing a page, use the `developer-docs-style` skill; for Mintlify mechanics, the `mintlify` skill.

## Mintlify llms.txt

https://www.mintlify.com/docs/llms.txt

## The API specs have upstreams

Each API tab is generated from a copy of a spec that lives in the service's own repo:

| Copy | Source of truth | Tab |
| --- | --- | --- |
| `api/public-api.yaml` | `public-api.yaml` in `noetive-semantik` | Semantik API |
| `api/bud-api.yaml` | `public-api.yaml` in `noetive-bud` | Bud API |
| `api/access-api.yaml` | `public-api.yaml` in `noetive-webapp` | Access API |

Sync is manual: re-copy the file whenever the service ships a contract change. A stale copy produces a silently wrong reference rather than a build failure, so nothing will warn you. Do not edit a copy — fix it upstream and re-copy, otherwise the next sync reverts the edit.

The Bud API tab lists its operations by hand, grouped by the spec's tags. A path added upstream does not appear until it is added to `docs.json`. `/health` and `/ready` in the Bud spec are left out on purpose, because they are for the load balancer and not for callers.

