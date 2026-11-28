# Paperclip hub inspection — 2026-11-28 (run 006, read-only)

Status: **OK**. The container is up, the server answers, and the state was read from the embedded Postgres in a `READ ONLY` transaction.
No container start/stop/restart, no DB writes, no POST/PUT/DELETE, no config edits, no `git checkout/restore`.

## 1. Container

Command: `docker ps --filter name=docker-paperclip-1`

```
CONTAINER ID   IMAGE              COMMAND                  CREATED        STATUS         PORTS                                         NAMES
a207b2409ea0   docker-paperclip   "/usr/bin/tini -s --…"   21 hours ago   Up 4 minutes   0.0.0.0:3100->3100/tcp, [::]:3100->3100/tcp   docker-paperclip-1
```

The container was created 21 hours ago and restarted about 4 minutes before this check (Nam's `up -d`).

## 2. Server

Commands:
```
curl -s -o /dev/null -m 10 -w "%{http_code}" http://localhost:3100/            -> 200
curl -s -o /dev/null -m 10 -w "%{http_code}" http://localhost:3100/api/health  -> 200
curl -s -m 10 http://localhost:3100/api/health
```
Body of `/api/health`:
`{"status":"ok","deploymentMode":"authenticated","deploymentExposure":"private","bootstrapStatus":"ready","databaseBackup":{"enabled":true,"status":"ok"},"commit":null,...}`

The earlier `000` was only the server still booting. It answers now. No log dig was needed. `docker logs docker-paperclip-1 --tail 50 | grep -ciE "error|fatal"` returned `0`.

## 3. Version

BLOCKED: the root `package.json` has **no `version` field** (it is `"name": "paperclip", "private": true`). `/api/health` reports `"commit": null`, so the build carries no version or commit stamp.
Source reference: this worktree is on branch `agent/20261128-006` at `85ad16b47`. Nam's commit `188429276` ("chore: khoa LF cho .sh/.sql ...") is on local `main` (`git branch -a --contains 188429276` → `main`), not on this branch. Which commit the running image was built from could not be proven.

## 4. Current state

### 4a. API (first attempt)
- `GET /api/companies` → **403**. BLOCKED: `deploymentMode: authenticated`, and an anonymous GET is refused. No credentials were used.
- `GET /api/agents` → 404 (not a top-level route).

### 4b. Fallback: embedded Postgres
- There is **no `psql` binary** in the container (`command -v psql` and `find / -name "psql*"` both return nothing).
- Postgres process: `.../@embedded-postgres/linux-x64/native/bin/postgres -D /paperclip/instances/default/db -p 54329`.
- Database `paperclip` on port 54329. The connection string is the built-in embedded default in `server/src/index.ts:744`; it is not from `docker/.env`.
- Client: the app's own `postgres` npm package (`/app/node_modules/.pnpm/postgres@3.4.9.../postgres`), run with `docker exec docker-paperclip-1 node -e ...`. Every query ran inside `sql.begin("read only", ...)`.
- The schema has 347 tables (listed via `information_schema.tables`).

Queries (non-sensitive columns only; `adapter_config`/`runtime_config` were read as **key names only**, never values):

```sql
select name,status,issue_prefix,budget_monthly_cents,spent_monthly_cents,require_board_approval_for_new_agents from companies;
select a.name,a.role,a.status,a.adapter_type,a.adapter_config->>'model' as model,a.budget_monthly_cents,a.spent_monthly_cents,a.lifecycle_state,c.name
  from agents a join companies c on c.id=a.company_id;
select jsonb_object_keys(adapter_config) from agents;   -- key names only
select count(*) from issues;
select status,count(*) from issues group by status;
select count(*) from issues where hidden_at is not null;
select identifier,title,status,priority,(assignee_agent_id is not null) from issues;
select count(*) from budget_policies;
select count(*) from cost_events;
```

### Company
| name | status | issue_prefix | budget_monthly_cents | spent_monthly_cents | board approval for new agents |
|---|---|---|---|---|---|
| F.U Company | active | FUC | 0 | 0 | false |

**Company count: 1**

### Agents: **1**
| name | role | status | adapter_type | model | budget_monthly_cents | lifecycle |
|---|---|---|---|---|---|---|
| F.U Agent | general | idle | claude_local | *(not set)* | 0 | ready |

`adapter_config` has no `model` key. Its keys are graceSec, timeoutSec, maxTurnsPerRun, paperclipSkillSync, instructions*, dangerouslySkipPermissions. So the agent uses the Claude CLI's default model. `runtime_config` keys: heartbeat, aiConnection.
Note: the `dangerouslySkipPermissions` key is present (value not read). Check it before giving this agent real work.

### Issues: total **1**
| status | count |
|---|---|
| in_review | 1 |

Hidden: 0. The one issue is "Chat with F.U Agent" (priority medium, assigned to an agent, identifier null). It is the test chat from 27/11.

### Budget / cost cap
- `companies.budget_monthly_cents = 0` and `agents.budget_monthly_cents = 0`. In this schema 0 means **no cap set**.
- `budget_policies`: **0 rows**, so there is no budget policy and no hard stop.
- `cost_events`: 2 rows (amounts not summed; spent_monthly_cents = 0).

**Conclusion: no budget or cost cap is set.**

## 5. Adapters in this build

Command: `ls packages/adapters`

```
claude-local  codex-local  cursor-cloud  cursor-local  gemini-local  grok-local
hermes  hermes-gateway  kimi-local  openclaw-gateway  opencode-local  pi-local
```
That is 12 adapter directories (plus `AUTHORING.md`).

## BLOCKED summary
- BLOCKED: version string. The root `package.json` has no `version` field and health reports `commit: null`.
- BLOCKED: reading state through the HTTP API. `/api/companies` returns 403 (authenticated mode). The DB fallback was used instead.
- BLOCKED (minor): `psql` is not installed in the container. The app's Node `postgres` client was used instead.
