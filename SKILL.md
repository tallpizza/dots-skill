---
name: dots
description: Dots is a graph application that lets users build, explore, and manage knowledge graphs through spaces. This skill provides the complete guide for interacting with Dots — reading graph snapshots, creating nodes and edges, running queries, executing templates, and controlling the shared browser view. Use this skill whenever the user mentions dots, graphs, nodes, edges, spaces, knowledge graphs, graph queries, templates, or wants to visualize, build, or explore any graph structure — even casual requests like "show me the graph", "add a node", or "what's connected to X".
---

# Dots

Use this skill when the user wants to read or mutate graph data through the Dots HTTP API.

## Workflow

### 1. Config resolution

Check for `dots.json` in CWD.

**If missing** — ask the user for API key (`sk_...` format) and base URL (default `http://localhost:8400`). Then discover available targets:

```bash
# List workspaces accessible with this key
curl -s -H "Authorization: Bearer $API_KEY" "$BASE_URL/api/workspaces"

# List spaces in a workspace
curl -s -H "Authorization: Bearer $API_KEY" "$BASE_URL/api/workspaces/$WS_SLUG/spaces"
```

Present the available workspaces and spaces. Let the user pick one (or match the one mentioned in their prompt). Then write `dots.json`:

```json
{
  "base_url": "http://localhost:8400",
  "workspace_slug": "acme",
  "space_slug": "demo",
  "api_key": "sk_..."
}
```

Add `dots.json` to `.gitignore` if not already present (contains API key).

**If exists** — read it and use the stored values.

### 2. Target resolution

- Default workspace/space come from `dots.json`.
- If the user names a different space in their prompt (e.g., "demo 스페이스에 넣어줘"), use that slug for the request. Update `dots.json` with the new space so it becomes the default for next time.
- If the user names an ambiguous or unknown space, fetch the list via `GET /api/workspaces/:ws/spaces` to resolve and confirm.

### 3. Execute

Make API calls using `curl` via Bash, reading credentials from `dots.json`.

### 4. Confirm

After mutations, fetch the graph snapshot or the specific node to confirm the result. There is no public `GET /edges/:edge_id` route, so confirm edge writes via the graph snapshot. Show a brief summary including affected IDs and the active space.

## Rules

- Credentials come from `dots.json` in CWD.
- Auth: `Authorization: Bearer sk_{public_id}_{secret}` on every request.
- Never invent endpoints outside this spec.
- Do not expose a live API key unless the user explicitly asks.
- Re-read `dots.json` before multi-step or destructive work if the target may have changed.

## Scopes

- `view`: graph get, node get, query, read templates, view snapshot, view stream
- `update`: all of `view` + node/edge CRUD, view events, bulk write templates

If the requested action conflicts with scope, say so plainly and stop.

## ID Format

ULID with 3-letter prefix: `nod_`, `edg_`, `evt_`

Pattern: `^[a-z]{3}_[0-9A-HJKMNP-TV-Z]{26}$`

## Endpoints

Base path: `/api/workspaces/:workspace_slug/spaces/:space_slug` (abbreviated `…/:sp`).

### Discovery

| Method | Path | Scope | Response |
|--------|------|-------|----------|
| GET | `/api/workspaces` | view | `Workspace[]` |
| GET | `/api/workspaces/:ws/spaces` | view | `Space[]` |

### Graph Snapshot

| Method | Path | Scope | Response |
|--------|------|-------|----------|
| GET | `…/:sp/graph` | view | `{ nodes: GNode[], edges: GEdge[] }` |

### Nodes

| Method | Path | Body | Scope | Response |
|--------|------|------|-------|----------|
| GET | `…/:sp/nodes/:node_id` | — | view | `GNode` |
| POST | `…/:sp/nodes` | `{ kind, title, props? }` | update | `GNode` (201) |
| PATCH | `…/:sp/nodes/:node_id` | `{ kind?, title?, props? }` | update | `GNode` |
| DELETE | `…/:sp/nodes/:node_id` | — | update | 204 (deletes incident edges) |

### Edges

| Method | Path | Body | Scope | Response |
|--------|------|------|-------|----------|
| POST | `…/:sp/edges` | `{ kind, from_node_id, to_node_id, props? }` | update | `GEdge` (201) |
| PATCH | `…/:sp/edges/:edge_id` | `{ kind?, props? }` | update | `GEdge` |
| DELETE | `…/:sp/edges/:edge_id` | — | update | 204 |

### Props Rules

- Keys: lowercase `^[a-z][a-z0-9_]*$`
- Values: `string | number | boolean | null | Array<Scalar>`
- Reserved (cannot use): `node_id`, `edge_id`, `kind`, `title`, `from_node_id`, `to_node_id`, `created_at`, `updated_at`, `created_by`, `id`, `space_id`

### Query

`POST …/:sp/query` (scope: `view`)

```json
{
  "root": { "node_ids": ["nod_..."], "kind": "concept" },
  "expand": [{ "dir": "out", "max_depth": 2 }],
  "where": [{ "field": "kind", "op": "eq", "value": "concept" }],
  "sort": { "field": "created_at", "order": "desc" },
  "page": { "limit": 50, "cursor": "..." },
  "return": { "nodes": true, "edges": true }
}
```

- `root` is required — must include `node_ids` or `kind` (or both).
- **Fields**: `node_id`, `kind`, `title`, `created_at`, `updated_at`
- **Operators**: `eq`, `in`, `contains`, `gt`, `lt`
- **Sort fields**: `title`, `created_at`, `updated_at`
- **Expand dir**: `in`, `out`, `both` (max_depth: 1–3)

Response: `{ nodes, edges, cursor? }`

### Templates

`POST …/:sp/templates/:template_id`

| Template | Params | Scope | Description |
|----------|--------|-------|-------------|
| `neighbors` | `{ node_id, depth: 1-3, limit }` | view | Adjacent nodes |
| `shortest_path` | `{ from_node_id, to_node_id, max_depth: 1-6 }` | view | Shortest path |
| `subgraph_extract` | `{ root_node_ids: [] (1-20), depth: 1-3 }` | view | Subgraph extraction |
| `degree_summary` | `{ node_ids?: [] (1-100), kind? }` | view | Degree analysis |
| `bulk_node_upsert` | `{ nodes: [{ kind, title, props? }, ...] }` (1-100) | update | Bulk node creation |
| `bulk_edge_upsert` | `{ edges: [{ kind, from_node_id, to_node_id, props? }, ...] }` (1-200) | update | Bulk edge creation |

Response: `{ template_id, scope, graph?, stats? }`

Notes:
- Bulk write template payloads are wrapped in `params`.
- Do not send server-owned fields like `space_id`, `created_by`, `created_at`, `updated_at`, `node_id`, or `edge_id` in bulk template input. The server fills those from the active route and authenticated actor.

### Shared View

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `…/:sp/view` | view | Current view snapshot |
| POST | `…/:sp/view/events` | update | Publish view event |
| GET | `…/:sp/view/stream` | view | SSE subscription |

**ViewSnapshot**:

```json
{
  "version": 1,
  "camera": { "mode": "fit_all" } | { "mode": "focus_node", "node_id": "nod_..." } | null,
  "selected_node_ids": [],
  "filters": [],
  "panel": null
}
```

**View events** (POST body — `event_id` must be `evt_` prefixed):

| type | payload |
|------|---------|
| `camera.focus_node` | `{ event_id, node_id }` |
| `camera.fit_all` | `{ event_id }` |
| `selection.set_nodes` | `{ event_id, node_ids }` |
| `filters.set` | `{ event_id, filters }` |
| `panel.open` | `{ event_id, panel }` |

**SSE stream events**: `server.connected`, `server.heartbeat` (10s), `view.event` (ViewBroadcast)

**ViewBroadcast**: `{ version, type, payload, actor, at }`

## Types

```typescript
type GNode = {
  node_id: string; space_id: string; kind: string; title: string; created_by: string
  props: Record<string, GraphValue>; created_at: string; updated_at: string
}

type GEdge = {
  edge_id: string; space_id: string; kind: string; from_node_id: string; to_node_id: string
  created_by: string; props: Record<string, GraphValue>; created_at: string; updated_at: string
}

type GraphValue = string | number | boolean | null | Array<string | number | boolean | null>
```

## Query Budget

- `expand.max_depth`: 3
- `page.limit`: max 200
- request body: max 256 KB
- response: max 1 MB
- query timeout: 3 000 ms
- template read timeout: 5 000 ms
- template write timeout: 10 000 ms
- max nodes in response: 500
- max edges in response: 1 000

## Mutation Rules

- Create nodes before dependent edges.
- Use PATCH for partial updates.
- Edge PATCH cannot change `from_node_id` or `to_node_id` — delete and recreate to re-link.
- Delete only the exact resource the user asked to remove.
- After writes, confirm the affected IDs and active space.
- Prefer `bulk_node_upsert` / `bulk_edge_upsert` templates for batch operations.

## Error Codes

| Code | Meaning |
|------|---------|
| `AUTH_REQUIRED` | No credentials provided |
| `AUTH_INVALID` | Invalid or expired key |
| `FORBIDDEN` | Scope too weak for the action |
| `RATE_LIMITED` | Too many requests |
| `INVALID_ID` | Malformed node/edge/event ID |
| `INVALID_KEY` | Malformed API key |
| `INVALID_GRAPH_PROPERTY` | Reserved property name in `props` |
| `BUDGET_EXCEEDED` | Query budget limit hit |
| `QUERY_BODY_TOO_LARGE` | Request body exceeds 256 KB |
| `QUERY_DEPTH_EXCEEDED` | Expand depth > 3 |
| `QUERY_LIMIT_EXCEEDED` | Page limit > 200 |
| `QUERY_RESULT_EXCEEDED` | Response exceeds node/edge caps |
| `QUERY_FIELD_DENIED` | Invalid field in where/sort |
| `QUERY_TIMEOUT` | Query took too long |
| `TEMPLATE_NOT_FOUND` | Unknown template ID |
| `VIEW_EVENT_DENIED` | Unknown view event type |
| `SPACE_NOT_READY` | Space is still provisioning |

Error response body: `{ request_id, code, message }`

HTTP status code is carried by the response itself, not duplicated in the JSON body.

## Lessons Learned

### Routes use slugs, not IDs

All public routes use `workspace_slug` + `space_slug` from `dots.json`, not internal IDs. Call `/api/workspaces/{workspace_slug}/spaces/{space_slug}/...` for all graph, query, template, and view requests.
