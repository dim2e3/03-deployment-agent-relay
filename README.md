# Agent Relay — Deployment

This is the deployment module submission for the AI Zoomcamp. It takes the
[Agent Relay](agent-relay/) starter — a small FastAPI service that lets
software agents register, hand each other tasks over HTTP, and report
results — from a SQLite-backed local starter to a Postgres-backed service
running on Kubernetes with a working CI/CD pipeline.

The application code itself is the course starter, unmodified in behavior;
everything below is the deployment path added on top of it: containerizing
it, porting its storage layer to PostgreSQL, deploying it to a local
Kubernetes cluster, and automating build/test/deploy in CI.

## What Agent Relay does

Agents register to get an identity and a bearer token, one agent sends
another a task (`POST /tasks`), the recipient claims it (`POST
/tasks/claim`), executes it, and reports a result (`POST
/tasks/{id}/complete`). The sender polls `GET /tasks/{id}` for the outcome.
Claims are leased with heartbeats and automatic recovery on lease expiry, so
a crashed worker doesn't leave a task stuck. Full protocol details are in
[`agent-relay/SPEC.md`](agent-relay/SPEC.md).

## What was added for this module

### 1. Docker

[`agent-relay/Dockerfile`](agent-relay/Dockerfile) builds the service on
`python:3.11-slim` with `uv`, and runs
`uvicorn main:app --host 0.0.0.0 --port 8000`. The explicit `--host 0.0.0.0`
matters: uvicorn defaults to binding `127.0.0.1`, which is unreachable from
outside the container even with a correct `-p` port mapping.

```bash
cd agent-relay
docker build -t agent-relay:local .
docker run -d --name agent-relay-local -p 8010:8000 agent-relay:local
```

### 2. PostgreSQL, via Docker Compose

The starter ships with SQLite and a single `BEGIN IMMEDIATE` writer
transaction to serialize claims (SQLite has no `FOR UPDATE SKIP LOCKED`).
[`agent-relay/database.py`](agent-relay/database.py) and
[`agent-relay/storage.py`](agent-relay/storage.py) were extended so the same
codebase runs against PostgreSQL, selected purely by `RELAY_DATABASE_URL`:

- `immediate_transaction()` only issues SQLite's `BEGIN IMMEDIATE` when the
  configured database is SQLite; on PostgreSQL it's an ordinary transaction.
- A new `lock_for_update()` helper adds `SELECT ... FOR UPDATE` on
  PostgreSQL (a no-op on SQLite). `claim_one` uses `SKIP LOCKED` so
  concurrent workers each grab a different queued task instead of blocking
  on one another; `heartbeat` and `commit_terminal` take a plain row lock so
  a race between heartbeat/complete/fail on the same task serializes
  correctly.
- `create_task` catches the unique-constraint violation a concurrent
  duplicate `Idempotency-Key` can raise under PostgreSQL's weaker isolation,
  and returns the row that won the race instead of erroring.

[`agent-relay/compose.yaml`](agent-relay/compose.yaml) runs the API
alongside a `postgres` service (named `postgres` in the compose file), with
a healthcheck gate (`depends_on: condition: service_healthy`) so the API
never starts before the database is ready, and a named volume for
persistence:

```bash
cd agent-relay
docker compose up -d --build
# dashboard: http://127.0.0.1:8010/
```

### 3. Kubernetes (kind)

[`agent-relay/k8s/`](agent-relay/k8s/) deploys the same two services to a
local [kind](https://kind.sigs.k8s.io/) cluster:

| File | Contents |
| --- | --- |
| `00-namespace.yaml` | the `agent-relay` namespace |
| `01-postgres-secret.yaml` | DB credentials and `RELAY_DATABASE_URL` |
| `02-postgres.yaml` | Postgres `Deployment` + `PersistentVolumeClaim` (1Gi, RWO) + `Service`, readiness/liveness via `pg_isready` |
| `03-agent-relay.yaml` | API `Deployment` + `Service`, readiness on `/ready`, liveness on `/health` |

Kubernetes has no equivalent of Compose's `depends_on: condition:
service_healthy`, so `03-agent-relay.yaml` adds an `initContainer` that
polls `pg_isready` against the `postgres` Service before the main container
starts — without it, a cold `kubectl apply` crash-loops the API pod until it
wins the startup race on its own.

```bash
kind create cluster --name agent-relay
docker build -t agent-relay:local agent-relay/
kind load docker-image agent-relay:local --name agent-relay
kubectl apply -f agent-relay/k8s/
kubectl -n agent-relay get pods    # both should reach 1/1 Running
kubectl -n agent-relay port-forward svc/agent-relay 8020:8000
# dashboard: http://127.0.0.1:8020/
```

### 4. CI/CD

[`agent-relay/.github/workflows/ci.yml`](agent-relay/.github/workflows/ci.yml)
runs on every push/PR to `main`:

1. **`test`** — spins up a `postgres:16-alpine` service container and runs
   the full `pytest` suite against it: the starter's protocol,
   sender/recipient access-boundary, concurrent-claim, lease-expiry, and
   dashboard-asset tests, plus
   `test_full_task_exchange_sender_sees_completed_status` (added this
   module — registers two agents, sends a task, claims and completes it,
   and asserts the sender observes `status: "completed"` with the correct
   output). Running this against real PostgreSQL exercises the
   `FOR UPDATE` / `SKIP LOCKED` locking path, not just SQLite's global
   writer lock.
2. **`build-and-deploy`** (`needs: test`, so it only runs if tests pass —
   if they fail, the job is skipped outright and whatever is already
   running in the cluster is left untouched) — builds the image with a
   timestamp-based unique tag (`kind` never re-pulls a tag it already has
   cached on a node, so reusing one static tag would silently fail to
   update the running pod), loads it into the kind cluster, applies the
   manifests, patches the Deployment to the new tag, waits for the rollout
   to finish, and smoke-tests `/ready` from inside the cluster.

The workflow was developed and run locally with
[`act`](https://nektosact.com/) against this machine's real Docker daemon
and a real local kind cluster — `act -j test` and
`act -j build-and-deploy` (or `act` for both) — before ever needing a push
to GitHub. Locally, `build-and-deploy` reuses whatever `agent-relay` kind
cluster is already running instead of creating a fresh one each time; on a
real GitHub-hosted runner (which can't reach a developer's machine) it
creates its own disposable cluster inside the job instead.

## Repository layout

```
03-deployment-agent-relay/
└── agent-relay/            # the course starter + everything above
    ├── main.py, storage.py, database.py, schemas.py, errors.py, worker.py
    ├── dashboard.html, dashboard.py
    ├── test_agent_relay.py
    ├── Dockerfile, .dockerignore
    ├── compose.yaml
    ├── k8s/
    └── .github/workflows/ci.yml
```

## Verifying end to end

Whichever way it's running (bare Docker, Compose, or kind), the same check
works: register two agents, send one a task, have the other claim and
complete it, and read the result back as the sender.

```bash
BASE=http://127.0.0.1:8010   # or :8020 for the kind port-forward

sender=$(curl -sS -X POST $BASE/api/v1/agents -d '{"name":"sender"}')
recipient=$(curl -sS -X POST $BASE/api/v1/agents -d '{"name":"echo"}')
# ...extract agent_id/token from each response, then:

curl -sS -X POST $BASE/api/v1/tasks \
  -H "Authorization: Bearer $SENDER_TOKEN" \
  -d '{"to":"'"$RECIPIENT_ID"'","input":"Reverse this string: agent relay"}'

curl -sS -X POST $BASE/api/v1/tasks/claim \
  -H "Authorization: Bearer $RECIPIENT_TOKEN" -d '{"worker_id":"demo"}'

curl -sS -X POST $BASE/api/v1/tasks/$TASK_ID/complete \
  -H "Authorization: Bearer $RECIPIENT_TOKEN" \
  -d '{"claim_token":"'"$CLAIM_TOKEN"'","output":"yaler tnega"}'

curl -sS $BASE/api/v1/tasks/$TASK_ID -H "Authorization: Bearer $SENDER_TOKEN"
# -> {"status":"completed","output":"yaler tnega",...}
```

Or open the dashboard directly and paste a token from any of the register
calls above into the "Agent token" field.
