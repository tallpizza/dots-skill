---
name: dots
description: Dots is a graph application that lets users build, explore, and manage knowledge graphs through spaces. This skill provides the guide for interacting with Dots over HTTP API — reading graph data, creating nodes and edges, running graph queries, and creating related nodes from an anchor node. Use this skill whenever the user mentions dots, graphs, nodes, edges, spaces, knowledge graphs, or graph queries.
---

# Dots

Use this skill when the user wants to read or mutate graph data through the Dots HTTP API.

## Workflow

### 1. Config resolution

Check for `dots.json` in CWD.

Default base URL is fixed to `https://dots.tallpizza.com`.

**If missing** — ask the user for API key (`sk_...` format). Only ask for a base URL if the user explicitly wants to use a different server. Then discover available targets:

```bash
# List workspaces accessible with this key
curl -s -H "Authorization: Bearer $API_KEY" "$BASE_URL/api/workspaces"

# List spaces in a workspace
curl -s -H "Authorization: Bearer $API_KEY" "$BASE_URL/api/workspaces/$WS_SLUG/spaces"
```

Present the available workspaces and spaces. Let the user pick one (or match the one mentioned in their prompt). Then write `dots.json`.

If the user did not explicitly override the base URL, omit `base_url` and rely on the default:

```json
{
  "workspace_slug": "acme",
  "space_slug": "demo",
  "api_key": "sk_..."
}
```

If the user explicitly chose a different server, include `base_url` in `dots.json`.

Add `dots.json` to `.gitignore` if not already present (contains API key).

**If exists** — read it and use the stored values.

### 2. Target resolution

- Default workspace/space come from `dots.json`.
- If the user names a different space in their prompt (e.g., "demo 스페이스에 넣어줘"), use that slug for the request. Update `dots.json` with the new space so it becomes the default for next time.
- If the user explicitly names a different base URL, use it and persist it to `dots.json`.
- If the user names an ambiguous or unknown space, fetch the list via `GET /api/workspaces/:ws/spaces` to resolve and confirm.

### 3. Execute

Make API calls using `curl` via Bash, reading credentials from `dots.json`.

### 4. Confirm

After mutations, fetch the graph snapshot or the specific node to confirm the result. There is no public `GET /edges/:edge_id` route, so confirm edge writes and snapshot restore results via the graph snapshot. Show a brief summary including affected IDs and the active space.

## Rules

- Credentials come from `dots.json` in CWD.
- Only persist `base_url` to `dots.json` when the user explicitly selected a non-default server.
- Auth: `Authorization: Bearer sk_{public_id}_{secret}` on every request.
- Never invent endpoints outside this spec.
- Do not expose a live API key unless the user explicitly asks.
- Re-read `dots.json` before multi-step or destructive work if the target may have changed.

## Scopes

- `view`: graph get, node get, query, snapshots list
- `update`: all of `view` + node/edge CRUD, related node creation, snapshots save/restore

If the requested action conflicts with scope, say so plainly and stop.

## ID Format

ULID with 3-letter prefix: `nod_`, `edg_`, `evt_`

Pattern: `^[a-z]{3}_[0-9A-HJKMNP-TV-Z]{26}$`

## Endpoints

Base path: `/api/workspaces/:workspace_slug/spaces/:space_slug` (abbreviated `…/:sp`).

### Discovery

`GET /api/workspaces`

- 설명: 현재 API key로 접근 가능한 workspace 목록 조회
- 사용: 처음 `dots.json`을 만들 때 workspace를 고를 때 사용
- 파라미터: 없음

`GET /api/workspaces/:ws/spaces`

- 설명: 특정 workspace 안의 space 목록 조회
- 사용: `workspace_slug`를 정한 뒤 `space_slug`를 고를 때 사용
- 파라미터:
  - `ws`: workspace slug

### Read APIs

`GET …/:sp/graph`

- 설명: 현재 space의 전체 그래프 스냅샷 조회
- 사용: 전체 노드/엣지를 한 번에 가져오고 싶을 때 사용
- 파라미터: 없음
- 응답: `{ nodes, edges }`

`POST …/:sp/query`

- 설명: 조건에 맞는 노드를 찾고 필요하면 주변 그래프로 확장해서 조회
- 사용:
  - `root`는 필수
  - `root`만 보내면 시작 노드 집합 조회
  - `root` + `where`면 시작 집합 안에서 조건 검색
  - `root` + `expand`면 시작 노드 주변 서브그래프 조회
- 파라미터:
  - `root`: 조회 시작점
    - `node_ids?`: 시작 노드 ID 배열
    - `kind?`: 시작 노드 kind
    - 둘 중 하나는 반드시 필요
  - `where?`: 조회 시작 조건 배열
    - `field`: `node_id` | `kind` | `title` | `created_at` | `updated_at`
    - `op`: `eq` | `in` | `contains` | `gt` | `lt`
    - `value`: scalar 또는 scalar 배열
  - `expand?`: 확장 규칙 배열
    - `dir`: `in` | `out` | `both`
    - `max_depth`: `1` | `2` | `3`
  - `sort?`: `{ field, order }`
    - `field`: `created_at` | `updated_at` | `title`
    - `order`: `asc` | `desc`
  - `page?`: `{ limit, cursor? }`
  - `return?`: `{ nodes, edges }`
- 응답: `{ nodes, edges, cursor? }`

`GET …/:sp/snapshots`

- 설명: 현재 space의 저장된 스냅샷 목록 조회
- 사용: 복원 가능한 시점을 확인할 때 사용
- 파라미터: 없음
- 권한: `view`
- 응답: `Snapshot[]`

```json
{
  "root": { "kind": "concept" },
  "where": [{ "field": "kind", "op": "eq", "value": "concept" }],
  "expand": [{ "dir": "out", "max_depth": 2 }],
  "page": { "limit": 50 },
  "return": { "nodes": true, "edges": true }
}
```

`GET …/:sp/nodes/:node_id`

- 설명: 단일 노드 상세 조회
- 사용: 특정 노드 하나의 최신 상태를 확인할 때 사용
- 파라미터:
  - `node_id`: 조회할 노드 ID
- 응답: `GNode`

`POST …/:sp/nodes/:node_id/related-words`

- 설명: 기준 노드에서 관련 단어 후보를 생성하거나, 바로 노드로 생성
- 사용:
  - `create_nodes`를 생략하면 기본값은 `true`
  - `create_nodes=false`: 미리보기만 조회
  - `create_nodes=true`: 관련 단어를 노드/엣지로 생성
- 파라미터:
  - `node_id`: 기준 노드 ID
  - `count?`: 생성할 항목 수
  - `create_nodes?`: `false`면 preview, `true`면 생성
- 응답:
  - preview: `{ created: false, items, nodes: [], edges: [] }`
  - create: `{ created: true, items, nodes, edges }`

### Write APIs

`POST …/:sp/nodes`

- 설명: 노드 생성
- 사용: 새 노드를 추가할 때 사용
- 파라미터:
  - `kind`: 노드 타입
  - `title`: 노드 제목
  - `props?`: 추가 속성
- 응답: `GNode`

`POST …/:sp/snapshots`

- 설명: 현재 space 상태를 저장된 스냅샷으로 생성
- 사용: 나중에 복원할 시점을 저장할 때 사용
- 파라미터:
  - `name?`: 스냅샷 이름
- 권한: `update`
- 응답: `{ snapshot: Snapshot }`

`PATCH …/:sp/nodes/:node_id`

- 설명: 노드 수정
- 사용: 노드의 `kind`, `title`, `props`를 부분 수정할 때 사용
- 파라미터:
  - `node_id`: 수정할 노드 ID
  - `kind?`, `title?`, `props?`, `mode?`
  - `mode`: `patch` 또는 `replace` (`props` 처리 방식, 기본값 `patch`)
- 응답: `GNode`

`DELETE …/:sp/nodes`

- 설명: 하나 이상의 노드 삭제
- 사용: 특정 노드와 그 incident edge를 제거하거나 여러 노드를 한 번에 제거할 때 사용
- 파라미터:
  - `id`: 삭제할 노드 ID 또는 ID 배열
- 응답: `204 No Content`

`POST …/:sp/edges`

- 설명: 엣지 생성
- 사용: 기존 두 노드를 연결할 때 사용
- 파라미터:
  - `kind`: 엣지 타입
  - `from_node_id`: 시작 노드 ID
  - `to_node_id`: 도착 노드 ID
  - `props?`: 추가 속성
- 응답: `GEdge`

`PATCH …/:sp/edges/:edge_id`

- 설명: 엣지 수정
- 사용: 엣지의 `kind` 또는 `props`를 수정할 때 사용
- 파라미터:
  - `edge_id`: 수정할 엣지 ID
  - `kind?`, `props?`, `mode?`
  - `mode`: `patch` 또는 `replace` (`props` 처리 방식, 기본값 `patch`)
- 응답: `GEdge`

`DELETE …/:sp/edges`

- 설명: 하나 이상의 엣지를 한 번에 삭제
- 사용: 여러 연결을 일괄 제거할 때 사용
- 파라미터:
  - `id`: 삭제할 엣지 ID 또는 ID 배열
- 응답: `204 No Content`

`POST …/:sp/linked-nodes`

- 설명: 새 노드 하나를 만들고 기존 노드 하나와 엣지 하나로 바로 연결
- 사용: 기존 노드에 새 노드 하나를 붙이면서 동시에 엣지도 만들 때 사용
- 파라미터:
  - `node`: 생성할 새 노드 정의
    - `kind`: 생성할 노드 타입
    - `title`: 생성할 노드 제목
    - `props?`: 생성 노드 props
  - `edge`: 새 노드를 기존 그래프에 연결할 엣지 정의
    - `kind`: 생성할 엣지 타입
    - `from_node_id`: 기존 시작 노드 ID
    - `props?`: 생성 엣지 props
- 응답: `{ node, edge }`

예시 (`POST …/:sp/linked-nodes`):

```json
{
  "node": {
    "kind": "concept",
    "title": "Graph database",
    "props": { "source": "ai" }
  },
  "edge": {
    "kind": "related_to",
    "from_node_id": "nod_01ARZ3NDEKTSV4RRFFQ69G5FAV",
    "props": { "weight": 1 }
  }
}
```

`POST …/:sp/snapshots/:snapshot_id/restore`

- 설명: 저장된 스냅샷 상태로 현재 space를 복원
- 사용: 이전 저장 시점으로 그래프를 되돌릴 때 사용
- 파라미터:
  - `snapshot_id`: 복원할 스냅샷 ID
- 권한: `update`
- 응답: `{ snapshot: Snapshot, restored: true }`

### Props Rules

- keys: lowercase `^[a-z][a-z0-9_]*$`
- values: `string | number | boolean | null | Array<Scalar>`
- reserved: `node_id`, `edge_id`, `kind`, `title`, `from_node_id`, `to_node_id`, `created_at`, `updated_at`, `created_by`, `id`, `space_id`

## Types

```typescript
type GNode = {
  node_id: string; kind: string; title: string; created_by: string
  props: Record<string, GraphValue>; created_at: string; updated_at: string
}

type GEdge = {
  edge_id: string; kind: string; from_node_id: string; to_node_id: string
  created_by: string; props: Record<string, GraphValue>; created_at: string; updated_at: string
}

type GraphValue = string | number | boolean | null | Array<string | number | boolean | null>

type Snapshot = {
  snapshot_id: string; name: string; node_count: number; edge_count: number
  created_by: string; created_at: string; updated_at: string
}
```

## Query Budget

- `expand.max_depth`: 3
- request body: max 256 KB
- response: max 1 MB
- query timeout: 3 000 ms
- max nodes in response: 500
- max edges in response: 1 000

## Mutation Rules

- Create nodes before dependent edges.
- Use PATCH for partial updates.
- Edge PATCH cannot change `from_node_id` or `to_node_id` — delete and recreate to re-link.
- Delete only the exact resource the user asked to remove.
- After writes, confirm the affected IDs and active space.
- Prefer `POST …/:sp/linked-nodes` when creating one new node already connected to an existing node.

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
| `QUERY_LIMIT_EXCEEDED` | Legacy query limit exceeded |
| `QUERY_RESULT_EXCEEDED` | Response exceeds node/edge caps |
| `QUERY_FIELD_DENIED` | Invalid field in `where` |
| `QUERY_TIMEOUT` | Query took too long |
| `AI_NOT_CONFIGURED` | Related-words AI backend is not configured |
| `SPACE_NOT_READY` | Space is still provisioning |

Error response body: `{ request_id, code, message }`

HTTP status code is carried by the response itself, not duplicated in the JSON body.

## Lessons Learned

### Routes use slugs, not IDs

All public routes use `workspace_slug` + `space_slug` from `dots.json`, not internal IDs. Call `/api/workspaces/{workspace_slug}/spaces/{space_slug}/...` for all graph, query, node, edge, and related-node requests.
