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

**If exists** — read it and **immediately proceed to Step 2 (Load space context) before doing anything else**. Do not start any node/edge work until the context check is complete.

### 2. Load space context (MANDATORY when `dots.json` exists)

`dots.json`이 존재하는 모든 세션에서 **사용자 작업을 시작하기 전에 반드시** 현재 space의 컨텍스트를 먼저 로드한다:

```bash
# Get space list (includes description for each space)
curl -s -H "Authorization: Bearer $API_KEY" "$BASE_URL/api/workspaces/$WS_SLUG/spaces"

# Get label definitions with descriptions
curl -s -H "Authorization: Bearer $API_KEY" "$BASE_URL/api/workspaces/$WS_SLUG/spaces/$SPACE_SLUG/labels"
```

#### 분기 처리

**Case A — space description과 label_defs가 모두 비어있음:**

space가 아직 정의되지 않은 상태이므로, 노드를 만들기 전에 사용자에게 먼저 space의 목적을 물어보고 라벨 정의를 함께 만든다.

```
이 space는 아직 description과 label 정의가 없어요. dots를 본격적으로 사용하기 전에 먼저:

1. 이 space의 목적이 뭔가요? (예: "decision tree", "knowledge base", "project tasks")
2. 어떤 종류의 노드를 다룰 건가요?

알려주시면 space description과 그에 맞는 label 정의(label + 각 description)를 먼저 만들고, 그 다음에 작업을 진행할게요.
```

사용자 답변을 받으면:
- `PATCH /api/workspaces/:ws/spaces/:space_slug` 로 space description 업데이트
- `POST /api/workspaces/:ws/spaces/:sp/labels` 로 라벨 정의 등록 (각 라벨의 `description` 포함)
- 결과를 사용자에게 확인받은 후에 본 작업 진행

**Case B — space description 또는 label_defs 중 하나라도 존재:**

기존 컨텍스트를 권위 있는 가이드로 받아들이고 그에 맞춰 작업한다:
- **Space description**: 해당 space의 목적, 용도, 사용 규칙. 노드를 생성하거나 구조를 결정할 때 이 설명을 따른다
- **Label descriptions**: 각 라벨의 사용 기준. 노드 생성 시 적절한 라벨을 선택하는 근거로 사용

새로 필요한 라벨이 있으면 사용자에게 확인 후 추가하고, 기존 라벨로 표현 가능하면 기존 것을 사용한다.

### 3. Target resolution

- Default workspace/space come from `dots.json`.
- If the user names a different space in their prompt (e.g., "demo 스페이스에 넣어줘"), use that slug for the request. Update `dots.json` with the new space so it becomes the default for next time. **새 space로 전환했다면 Step 2를 그 space에 대해 다시 실행한다.**
- If the user explicitly names a different base URL, use it and persist it to `dots.json`.
- If the user names an ambiguous or unknown space, fetch the list via `GET /api/workspaces/:ws/spaces` to resolve and confirm.

### 4. Execute

Make API calls using `curl` via Bash, reading credentials from `dots.json`.

### 5. Confirm

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

- 설명: 특정 workspace 안의 space 목록 조회. 각 space에는 `description` 필드가 포함됨
- 사용: `workspace_slug`를 정한 뒤 `space_slug`를 고를 때 사용
- 파라미터:
  - `ws`: workspace slug
- 응답: 각 space 객체에 `id`, `slug`, `name`, `description`, `label_defs` 등 포함

### Read APIs

`GET …/:sp/labels`

- 설명: 현재 space에 정의된 노드 라벨 목록과 설명 조회
- 사용: AI나 사용자가 이 space에서 허용되는 노드 라벨을 확인할 때 사용
- 파라미터: 없음
- 응답: `Array<{ label, description }>`

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
    - `kind?`: 시작 노드 kind (`deprecated`, 내부적으로 label로 정규화됨)
    - `label?`: 시작 노드 label (`권장`)
    - `node_ids`, `kind`, `label` 중 하나는 반드시 필요
  - `where?`: 조회 시작 조건 배열
    - `field`: `node_id` | `kind` | `label` | `title` | `created_at` | `updated_at`
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

### Space Management

`POST /api/workspaces/:ws/spaces`

- 설명: 새 space 생성
- 사용: workspace에 새 space를 만들 때 사용
- 파라미터:
  - `name`: space 이름
  - `slug`: space slug (lowercase, numbers, hyphens)
  - `description?`: space 설명 — 이 space의 목적, 사용법, 라벨 규칙 등을 기술
- 권한: workspace admin
- 응답: `{ space }`

`PATCH /api/workspaces/:ws/spaces/:space_slug`

- 설명: space 메타데이터 수정
- 사용: space의 이름, slug, 설명을 변경할 때 사용
- 파라미터:
  - `name`: space 이름
  - `slug`: space slug
  - `description?`: space 설명
- 권한: workspace admin
- 응답: `{ space }`

`POST …/:sp/labels`

- 설명: space 라벨 정의 추가
- 사용: 허용 라벨과 설명을 새로 등록할 때 사용
- 파라미터:
  - `label`: 라벨 이름
  - `description?`: 라벨 설명
- 응답: `{ label }`

`PATCH …/:sp/labels/:label`

- 설명: space 라벨 정의 수정
- 사용: 라벨 이름 또는 설명을 수정할 때 사용
- 파라미터:
  - `label`: 현재 라벨 이름(path)
  - body: `label?`, `description?`
- 응답: `{ label }`

`DELETE …/:sp/labels/:label`

- 설명: space 라벨 정의 삭제
- 사용: 허용 라벨 목록에서 제거할 때 사용
- 파라미터:
  - `label`: 삭제할 라벨 이름(path)
- 응답: `{ ok: true }`

### Write APIs

`POST …/:sp/nodes`

- 설명: 노드 생성
- 사용: 새 노드를 추가할 때 사용
- 파라미터:
  - `kind?`: 노드 타입 (`labels`가 있으면 생략 가능)
  - `labels?`: 노드 라벨 배열 (권장)
  - `title?`: 노드 제목. 생략하면 `props.title` 또는 `content` 첫 비어있지 않은 줄에서 유도됨
  - `content?`: 노드 본문 마크다운
  - `props?`: 추가 속성 (`content`는 여기에 넣지 말고 top-level `content` 사용)
- 규칙: space에 label definition이 하나 이상 있으면 `labels`는 반드시 정의된 라벨만 사용
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
- 사용: 노드의 `labels`, `content`, `props`를 중심으로 부분 수정할 때 사용 (`kind`, `title`는 legacy compatibility)
- 파라미터:
  - `node_id`: 수정할 노드 ID
  - `kind?`, `labels?`, `title?`, `content?`, `props?`, `mode?`
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
- `content` is the main node body and should be sent as top-level `content`, not inside `props`
- `labels` are the canonical node type field; `kind` is deprecated legacy compatibility and should be treated as derived from the first label
- `props.title` is allowed and used as the structured title field
- UI captions/titles should prefer the first non-empty line of `content`, then fall back to `props.title`, then other legacy title fields
- reserved: `node_id`, `edge_id`, `kind`, `from_node_id`, `to_node_id`, `created_at`, `updated_at`, `created_by`, `id`, `space_id`
- convention: node content is the main body and supports markdown. Store it in `props.content` for UI editing, while the HTTP API remains the same (`props` is still just a key-value object). Existing clients using other props continue to work.
- UI convention: when a graph view or picker needs a node caption, prefer the first non-empty line of `props.content`; fall back to `title` if content is empty.

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

- **MANDATORY first step on every session with `dots.json`**: read the space's `description` (from `GET /api/workspaces/:ws/spaces`) and `GET …/:sp/labels`. The space description defines the space's purpose and conventions; the label descriptions define when each label should be used. Treat these as authoritative instructions for how to structure nodes in this space.
- If both space description and label_defs are empty, **do not create any nodes yet** — first ask the user about the space's purpose, then write the space description and create label definitions (label + description). Only after these are in place should node creation proceed.
- If the space has one or more label definitions, create/update nodes using only those labels.
- If the user asks for a new label in that case, create it first via the labels CRUD endpoints or ask for confirmation.
- Do not rename or delete a defined label that is still used by existing nodes; report the conflict plainly.
- Create nodes before dependent edges.
- Use PATCH for partial updates.
- Edge PATCH cannot change `from_node_id` or `to_node_id` — delete and recreate to re-link.
- Delete only the exact resource the user asked to remove.
- After writes, confirm the affected IDs and active space.
- Prefer `POST …/:sp/linked-nodes` when creating one new node already connected to an existing node.
- Treat node `content` as the primary note body, and support markdown in it.
- When the user wants to write rich notes/content on a node, store the markdown string in `props.content` instead of inventing a new top-level field.
- For display, prefer the first non-empty line of `props.content` as the node caption/title in UI contexts.

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
| `SPACE_NOT_READY` | Space is still provisioning |

Error response body: `{ request_id, code, message }`

HTTP status code is carried by the response itself, not duplicated in the JSON body.

## Lessons Learned

### Routes use slugs, not IDs

All public routes use `workspace_slug` + `space_slug` from `dots.json`, not internal IDs. Call `/api/workspaces/{workspace_slug}/spaces/{space_slug}/...` for all graph, query, node, edge, and related-node requests.
