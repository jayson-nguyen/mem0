# Mem0 Remote Database Connection Guide (Progressive)

This guide is intentionally written step-by-step.
Only the current step is documented now. I will append the next step after you send results.

## Step 1: Confirm Network Path and Remote Service Reachability

Goal:
- Verify how your local machine can reach the company server.
- Verify Postgres and Qdrant are actually reachable from where they are hosted.
- Collect the minimum connection details we need before configuring Mem0.

Known remote container ports (from your note):
- Postgres: `9045 -> 5432`
- Qdrant HTTP: `9033 -> 6333`
- Qdrant gRPC: `9034 -> 6334`

---

### 1A) Run checks on the company server

Run these on the company server shell.

```bash
# 1) Confirm both containers are up and port mappings are present
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}\t{{.Names}}' | grep -E 'postgres|qdrant|CONTAINER'

# 2) Confirm host is listening on expected published ports
ss -ltnp | grep -E ':(9045|9033|9034)\b'

# 3) Quick HTTP health check for Qdrant from the server host
curl -i http://127.0.0.1:9033/

# 4) Optional: show if firewall rules may block these ports
# Ubuntu/Debian with UFW
sudo ufw status verbose || true

# RHEL/CentOS with firewalld
sudo firewall-cmd --list-ports || true
```

What to check and report back:
- Command (1): both postgres and qdrant lines are present and status is Up.
- Command (2): listeners exist on 0.0.0.0:9045 and 0.0.0.0:9033 (and usually 9034).
- Command (3): returns HTTP response (200/404 is fine, timeout/refused is not fine).
- Command (4): whether firewall appears to block inbound 9045/9033/9034.

---

### 1B) Run checks on your local machine

Run these on your local machine terminal.

```bash
# Set this first
export COMPANY_SERVER_HOST="<server-ip-or-dns>"

# 1) Basic network reachability
ping -c 3 "$COMPANY_SERVER_HOST"

# 2) Test direct TCP access to published DB ports
nc -vz "$COMPANY_SERVER_HOST" 9045
nc -vz "$COMPANY_SERVER_HOST" 9033
nc -vz "$COMPANY_SERVER_HOST" 9034

# 3) Test Qdrant HTTP endpoint directly
curl -i "http://$COMPANY_SERVER_HOST:9033/"
```

If `nc` is not installed locally:

```bash
# Debian/Ubuntu
sudo apt-get update && sudo apt-get install -y netcat-openbsd

# RHEL/CentOS
sudo yum install -y nc
```

What to check and report back:
- `ping`: whether host resolves/replies (if blocked by policy, that is okay, just note it).
- `nc` for 9045/9033/9034: `succeeded` or `open` means direct access works.
- `curl`: HTTP response means Qdrant is reachable directly.

---

### 1C) Provide required connection inputs (no secrets in chat if you prefer)

Please report these items (mask secrets if needed):

1. `COMPANY_SERVER_HOST` value.
2. Whether direct access from local machine works for:
   - 9045
   - 9033
   - 9034
3. SSH path details:
   - direct SSH to server, or
   - SSH via bastion/jump host.
4. Postgres details for Mem0:
   - DB name
   - DB user
   - Password available (yes/no)
5. Qdrant auth:
   - API key required (yes/no)

Expected decision after Step 1:
- If direct port access works and is allowed by policy, we can connect directly.
- If direct access fails or is disallowed, Step 2 will use SSH tunneling.

Step 1 outcome from your report:
- Direct access works from local machine to server on 9045, 9033, and 9034.
- Qdrant HTTP endpoint is reachable from both server host and local machine.
- Skipping optional sudo firewall commands is acceptable for now.

## Step 2: Configure Local Mem0 to Use Remote Postgres and Qdrant

Goal:
- Keep Mem0 running locally, but point it to the company server databases.
- Validate connectivity from local Docker containers (same path used by mem0-api container).

Decision for this run:
- Use direct connection mode (no SSH tunnel), based on Step 1 results.

---

### 2A) Prepare variables on your local machine

Run these on your local machine terminal.

```bash
cd /home/nmtri2110/Workspace/TVPL-AI/mem0

# From your Step 1 report
export COMPANY_SERVER_HOST="14.161.5.155"
export COMPANY_POSTGRES_PORT="9045"
export COMPANY_QDRANT_HTTP_PORT="9033"
export COMPANY_QDRANT_GRPC_PORT="9034"

# Fill with your real Postgres credentials for the company server
read -rp "POSTGRES DB user: " MEM0_DB_USER
read -rp "POSTGRES DB name: " MEM0_DB_NAME
read -rsp "POSTGRES DB password: " MEM0_DB_PASSWORD
echo
```

What to check and report back:
- Confirm you have values for DB user, DB name, and DB password.

---

### 2B) Update .ops/.env to remote DB endpoints (local machine)

Run these on your local machine terminal.

```bash
cd /home/nmtri2110/Workspace/TVPL-AI/mem0

# 1) Backup current env file first
cp .ops/.env ".ops/.env.backup.$(date +%Y%m%d_%H%M%S)"

# 2) URL-encode Postgres password for DATABASE_URL safety
export ENCODED_DB_PASSWORD="$(python3 - <<'PY'
import os
from urllib.parse import quote
print(quote(os.environ['MEM0_DB_PASSWORD'], safe=''))
PY
)"

# 3) Rewrite env vars used by compose/services
sed -i "s|^SHARED_POSTGRES_HOST=.*|SHARED_POSTGRES_HOST=${COMPANY_SERVER_HOST}|" .ops/.env
sed -i "s|^SHARED_POSTGRES_PORT=.*|SHARED_POSTGRES_PORT=${COMPANY_POSTGRES_PORT}|" .ops/.env
sed -i "s|^SHARED_QDRANT_HOST=.*|SHARED_QDRANT_HOST=${COMPANY_SERVER_HOST}|" .ops/.env
sed -i "s|^SHARED_QDRANT_PORT=.*|SHARED_QDRANT_PORT=${COMPANY_QDRANT_HTTP_PORT}|" .ops/.env

sed -i "s|^DATABASE_URL=.*|DATABASE_URL=postgresql://${MEM0_DB_USER}:${ENCODED_DB_PASSWORD}@${COMPANY_SERVER_HOST}:${COMPANY_POSTGRES_PORT}/${MEM0_DB_NAME}|" .ops/.env
sed -i "s|^QDRANT_HOST=.*|QDRANT_HOST=${COMPANY_SERVER_HOST}|" .ops/.env
sed -i "s|^QDRANT_PORT=.*|QDRANT_PORT=${COMPANY_QDRANT_HTTP_PORT}|" .ops/.env

# 4) Show sanitized values (no raw password)
grep -E '^(SHARED_POSTGRES_HOST|SHARED_POSTGRES_PORT|SHARED_QDRANT_HOST|SHARED_QDRANT_PORT|QDRANT_HOST|QDRANT_PORT)=' .ops/.env
grep '^DATABASE_URL=' .ops/.env | sed -E 's#(postgresql://[^:]+:)[^@]+(@.*)#\1***\2#'
```

What to check and report back:
- The printed values show host 14.161.5.155 and ports 9045/9033.
- Sanitized DATABASE_URL looks correct (user + host + port + db name).

---

### 2C) Validate remote DB reachability from Docker containers (local machine)

Run these on your local machine terminal.

```bash
cd /home/nmtri2110/Workspace/TVPL-AI/mem0

# Ensure external network declared in compose exists locally
docker network inspect shared_service_network >/dev/null 2>&1 || docker network create shared_service_network

# 1) TCP checks from inside a container
docker run --rm busybox:1.36 sh -c "nc -zvw5 ${COMPANY_SERVER_HOST} ${COMPANY_POSTGRES_PORT} && nc -zvw5 ${COMPANY_SERVER_HOST} ${COMPANY_QDRANT_HTTP_PORT} && nc -zvw5 ${COMPANY_SERVER_HOST} ${COMPANY_QDRANT_GRPC_PORT}"

# 2) Qdrant HTTP check from inside a container
docker run --rm curlimages/curl:8.7.1 -sS -i "http://${COMPANY_SERVER_HOST}:${COMPANY_QDRANT_HTTP_PORT}/" | sed -n '1,10p'

# 3) Postgres authentication check from inside a container
docker run --rm -e PGPASSWORD="$MEM0_DB_PASSWORD" postgres:13-alpine \
   psql -h "$COMPANY_SERVER_HOST" -p "$COMPANY_POSTGRES_PORT" -U "$MEM0_DB_USER" -d "$MEM0_DB_NAME" -c "select now();"
```

What to check and report back:
- Command (1): all three nc checks succeed.
- Command (2): HTTP response from Qdrant (200 expected).
- Command (3): psql returns one row for select now().

If command (3) fails:
- Report exact error text. Common causes are wrong credentials, wrong DB name, or server-side pg_hba/network policy.

---

### 2D) What to send me after Step 2

Please send:

1. Sanitized output from Step 2B (especially DATABASE_URL masked line).
2. Full output from Step 2C command (1), (2), and (3).
3. If any command fails, include exact error text and which environment it was run on (local machine).

After that, I will append Step 3 to start Mem0 stack and perform write/read verification against both Postgres and Qdrant.

---

### 2E) Fix Postgres DB name mismatch (local machine)

Why you are failing now:
- `psql: FATAL: database "mem0_openmemory" does not exist` means network/auth reached PostgreSQL successfully, but the database name in `DATABASE_URL` is not present on the server.

Observed in this run (from current `.ops/.env` credentials):
- Existing DBs: `airflow`, `langfuse`, `postgres`, `tvpl_ai`
- Missing DB: `mem0_openmemory`

Run these on your local machine terminal.

```bash
cd /home/nmtri2110/Workspace/TVPL-AI/mem0

# 1) Read current target DB from .ops/.env (masked output)
grep '^DATABASE_URL=' .ops/.env | sed -E 's#(postgresql://[^:]+:)[^@]+(@[^/]+/)([^?]+).*#\1***\2\3#'

# 2) List actual databases on remote server using same credentials (connect to postgres DB)
export DATABASE_URL="$(grep '^DATABASE_URL=' .ops/.env | cut -d= -f2-)"
export POSTGRES_URL="${DATABASE_URL%/*}/postgres"

docker run --rm postgres:13-alpine \
   psql -w "$POSTGRES_URL" -Atc "select datname from pg_database where datistemplate=false order by 1;"
```

Choose one fix path:

```bash
# Path A (quick unblock): use an existing DB name from the list above
read -rp "Use existing DB name: " MEM0_DB_NAME

# Replace only the trailing DB name in DATABASE_URL
sed -i -E "s#^(DATABASE_URL=postgresql://[^/]+/)[^?]+#\1${MEM0_DB_NAME}#" .ops/.env

# Show masked result
grep '^DATABASE_URL=' .ops/.env | sed -E 's#(postgresql://[^:]+:)[^@]+(@.*)#\1***\2#'
```

```bash
# Path B (preferred): keep mem0_openmemory and create it (requires CREATE DATABASE privilege)
docker run --rm postgres:13-alpine \
   psql -w "$POSTGRES_URL" -c "CREATE DATABASE mem0_openmemory;"

# If error says permission denied, ask DBA to run it on the company server.
```

Re-run Postgres validation (same environment: local machine):

```bash
cd /home/nmtri2110/Workspace/TVPL-AI/mem0

# Reload DATABASE_URL after edits
export DATABASE_URL="$(grep '^DATABASE_URL=' .ops/.env | cut -d= -f2-)"

docker run --rm postgres:13-alpine \
   psql -w "$DATABASE_URL" -c "select now();"
```

What to check and report back:
- `select now();` returns one row.
- No `database does not exist` error.
- Send masked `DATABASE_URL` line after the fix.

## Step 3: Start Local Mem0 and Verify End-to-End Persistence

Goal:
- Run Mem0 locally (API + UI) while using remote Postgres and Qdrant.
- Verify API/UI health.
- Write one test memory and read it back.
- Confirm persistence in Postgres and Qdrant.

Where to run Step 3:
- All commands in Step 3 are run on your local machine.
- No server-shell commands required for this step.

What Step 3 validates:
- Runtime connectivity from `mem0-api` container to remote Postgres + Qdrant.
- OpenMemory API works at `http://localhost:9100`.
- OpenMemory UI works at `http://localhost:9101`.
- New memory write is visible through API read.
- Postgres row count increases and Qdrant point totals are visible.

---

### 3A) Pre-flight: Compose command availability (local machine)

Run these on your local machine terminal.

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0

# Detect which compose command works
if docker compose version >/dev/null 2>&1; then
   COMPOSE_CMD="docker compose"
elif docker-compose version >/dev/null 2>&1; then
   COMPOSE_CMD="docker-compose"
else
   echo "No working Docker Compose detected."
   echo "Ubuntu/Debian fix: sudo apt-get update && sudo apt-get install -y docker-compose-plugin"
   exit 1
fi

echo "Using compose command: $COMPOSE_CMD"
```

What to check:
- A working compose command is selected.
- If not, install compose plugin first and rerun 3A.

---

### 3B) Start Mem0 stack (local machine)

Run these on your local machine terminal.

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0

# Ensure shared network exists
docker network inspect shared_service_network >/dev/null 2>&1 || docker network create shared_service_network

# Start stack
$COMPOSE_CMD -f .ops/docker-compose.mem0-shared.yml --project-directory .ops up -d

# Check container status
$COMPOSE_CMD -f .ops/docker-compose.mem0-shared.yml --project-directory .ops ps
```

What to check:
- `shared-services-ready`, `mem0-api`, and `mem0-ui` are `Up` (or healthy).

If Step 3B fails with `dependency failed to start: container mem0-shared-mem0-api-1 is unhealthy`:

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0

# Inspect exact mem0-api failure
docker compose -f .ops/docker-compose.mem0-shared.yml --project-directory .ops logs mem0-api --tail=220
docker inspect mem0-shared-mem0-api-1 --format '{{json .State.Health}}'
```

Known failure pattern in this run and fix:
- Error in logs: `invalid dsn: invalid connection option "check_same_thread"`
- Meaning: API process passed a SQLite-only option while using PostgreSQL, so API never booted.
- Fix already applied in this workspace:
   - `openmemory/api/app/database.py` now applies `check_same_thread` only for SQLite URLs.
   - `.ops/docker-compose.mem0-shared.yml` mounts local API code and local `mem0` package into `mem0-api`.
   - `.ops/docker-compose.mem0-shared.yml` uses image-compatible healthchecks:
      - `mem0-api`: Python `urllib` probe on `127.0.0.1:8765/docs`
      - `mem0-ui`: `wget` probe on `127.0.0.1:3000`

Recovery commands (local machine):

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0

docker rm -f mem0-shared-mem0-api-1 mem0-shared-mem0-ui-1 mem0-shared-shared-services-ready-1 2>/dev/null || true
docker compose -f .ops/docker-compose.mem0-shared.yml --project-directory .ops up -d
docker compose -f .ops/docker-compose.mem0-shared.yml --project-directory .ops ps
```

Expected healthy state after recovery:
- `mem0-shared-shared-services-ready-1`: healthy
- `mem0-shared-mem0-api-1`: healthy
- `mem0-shared-mem0-ui-1`: healthy

---

### 3C) Health checks for API and UI (local machine)

Run these on your local machine terminal.

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0

curl -sS -i http://localhost:9100/docs | sed -n '1,10p'
curl -sS -i http://localhost:9101/ | sed -n '1,10p'

export MEM0_USER_ID="$(grep '^MEM0_USER_ID=' .ops/.env | cut -d= -f2-)"
curl -sS "http://localhost:9100/api/v1/stats/?user_id=${MEM0_USER_ID}"
```

What to check:
- API `/docs` returns HTTP response.
- UI root returns HTTP response.
- Stats endpoint returns JSON (not connection/auth errors).

---

### 3D) Capture baseline counts before write (local machine)

Run these on your local machine terminal.

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0

export DATABASE_URL="$(grep '^DATABASE_URL=' .ops/.env | cut -d= -f2-)"
export QDRANT_HOST="$(grep '^QDRANT_HOST=' .ops/.env | cut -d= -f2-)"
export QDRANT_PORT="$(grep '^QDRANT_PORT=' .ops/.env | cut -d= -f2-)"

[ -n "$QDRANT_HOST" ] || { echo "QDRANT_HOST is empty; check .ops/.env"; exit 1; }
[ -n "$QDRANT_PORT" ] || { echo "QDRANT_PORT is empty; check .ops/.env"; exit 1; }

# Postgres baseline
docker run --rm postgres:13-alpine psql -w "$DATABASE_URL" -Atc "select count(*) from memories;" | tee /tmp/mem0_step3_pg_before.txt

# Qdrant baseline
python3 - <<'PY' | tee /tmp/mem0_step3_qdrant_before.txt
import json
import os
import urllib.request

base = f"http://{os.environ['QDRANT_HOST']}:{os.environ['QDRANT_PORT']}"

def get(path: str):
      with urllib.request.urlopen(base + path, timeout=15) as r:
            return json.load(r)

root = get("/collections")
collections = [c["name"] for c in root.get("result", {}).get("collections", [])]
print(f"TOTAL_COLLECTIONS={len(collections)}")
total_points = 0
for name in collections:
      info = get(f"/collections/{name}").get("result", {})
      points = info.get("points_count")
      vectors = info.get("vectors_count")
      if isinstance(points, int):
            total_points += points
      print(f"{name}\tpoints_count={points}\tvectors_count={vectors}")
print(f"TOTAL_POINTS={total_points}")
PY
```

What to check:
- Baseline files are created under `/tmp/`.

Common mistakes to avoid in 3D:
- If you get `KeyError: 'QDRANT_HOST'`, you likely forgot `export QDRANT_HOST=...`.
- If Python shows strange text around `PY` (for example `PYint(...)`), the heredoc was pasted incorrectly. Re-copy the whole block from ``python3 - <<'PY'`` down to the closing `PY` on its own line.

---

### 3E) Create and read one memory via API (local machine)

Run these on your local machine terminal.

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0

export MEM0_API_URL="http://localhost:9100"
export MEM0_USER_ID="$(grep '^MEM0_USER_ID=' .ops/.env | cut -d= -f2-)"
export STEP3_TAG="STEP3_REMOTE_DB_TEST_$(date +%s)"

# Create memory (infer=false keeps this test focused on connectivity/persistence)
curl -sS -X POST "$MEM0_API_URL/api/v1/memories/" \
   -H 'Content-Type: application/json' \
   -d "{\"user_id\":\"${MEM0_USER_ID}\",\"text\":\"${STEP3_TAG}: end-to-end remote DB test\",\"metadata\":{\"source\":\"step3\",\"tag\":\"${STEP3_TAG}\"},\"infer\":false,\"app\":\"openmemory\"}" \
   | tee /tmp/mem0_step3_create.json

export CREATED_MEMORY_ID="$(python3 - <<'PY'
import json
obj = json.load(open('/tmp/mem0_step3_create.json'))
print(obj.get('id', ''))
PY
)"

if [ -z "$CREATED_MEMORY_ID" ]; then
   echo "Create response did not return memory id. See /tmp/mem0_step3_create.json"
   exit 1
fi

echo "CREATED_MEMORY_ID=$CREATED_MEMORY_ID"

# Read memory by ID
curl -sS "$MEM0_API_URL/api/v1/memories/${CREATED_MEMORY_ID}" | tee /tmp/mem0_step3_get_by_id.json

# Query list by tag
curl -sS "$MEM0_API_URL/api/v1/memories/?user_id=${MEM0_USER_ID}&search_query=${STEP3_TAG}&page=1&size=20" | tee /tmp/mem0_step3_search.json
```

What to check:
- Create response returns `id`.
- `get by id` returns the same memory content/tag.
- Search/list response includes the new memory.

If create step returns an error, use these fixes:

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0

# A) If you see: "404 page not found"
#    Cause: config stored in DB still points to default OpenAI models not available on your endpoint.
curl -sS -X PUT http://localhost:9100/api/v1/config/mem0/llm \
   -H 'Content-Type: application/json' \
   -d '{"provider":"openai","config":{"model":"meta/llama-3.3-70b-instruct","temperature":0.1,"max_tokens":2000,"api_key":"env:OPENAI_API_KEY"}}'

curl -sS -X PUT http://localhost:9100/api/v1/config/mem0/embedder \
   -H 'Content-Type: application/json' \
   -d '{"provider":"openai","config":{"model":"nvidia/nv-embed-v1","api_key":"env:OPENAI_API_KEY"}}'

# B) If you see: "Vector dimension error: expected dim: 1536, got 4096"
#    Cause: existing Qdrant collection was created with 1536-d vectors.
export QDRANT_HOST="$(grep '^QDRANT_HOST=' .ops/.env | cut -d= -f2-)"
export QDRANT_PORT="$(grep '^QDRANT_PORT=' .ops/.env | cut -d= -f2-)"

curl -sS -X PUT http://localhost:9100/api/v1/config/mem0/vector_store \
   -H 'Content-Type: application/json' \
   -d "{\"provider\":\"qdrant\",\"config\":{\"host\":\"${QDRANT_HOST}\",\"port\":${QDRANT_PORT},\"collection_name\":\"openmemory_mem0_test\",\"embedding_model_dims\":4096}}"
```

---

### 3F) Verify persistence after write (local machine)

Run these on your local machine terminal.

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0

export DATABASE_URL="$(grep '^DATABASE_URL=' .ops/.env | cut -d= -f2-)"
export QDRANT_HOST="$(grep '^QDRANT_HOST=' .ops/.env | cut -d= -f2-)"
export QDRANT_PORT="$(grep '^QDRANT_PORT=' .ops/.env | cut -d= -f2-)"

# Postgres after-write count + direct row verification
docker run --rm postgres:13-alpine psql -w "$DATABASE_URL" -Atc "select count(*) from memories;" | tee /tmp/mem0_step3_pg_after.txt
docker run --rm postgres:13-alpine psql -w "$DATABASE_URL" -c "select id, content, created_at from memories where id='${CREATED_MEMORY_ID}';"

# Qdrant after-write snapshot
python3 - <<'PY' | tee /tmp/mem0_step3_qdrant_after.txt
import json
import os
import urllib.request

base = f"http://{os.environ['QDRANT_HOST']}:{os.environ['QDRANT_PORT']}"

def get(path: str):
      with urllib.request.urlopen(base + path, timeout=15) as r:
            return json.load(r)

root = get("/collections")
collections = [c["name"] for c in root.get("result", {}).get("collections", [])]
print(f"TOTAL_COLLECTIONS={len(collections)}")
total_points = 0
for name in collections:
      info = get(f"/collections/{name}").get("result", {})
      points = info.get("points_count")
      vectors = info.get("vectors_count")
      if isinstance(points, int):
            total_points += points
      print(f"{name}\tpoints_count={points}\tvectors_count={vectors}")
print(f"TOTAL_POINTS={total_points}")
PY

echo "Postgres count before: $(cat /tmp/mem0_step3_pg_before.txt)"
echo "Postgres count after : $(cat /tmp/mem0_step3_pg_after.txt)"

grep '^TOTAL_POINTS=' /tmp/mem0_step3_qdrant_before.txt /tmp/mem0_step3_qdrant_after.txt
```

What to check:
- Postgres count after-write is greater than or equal to before-write.
- Row with `CREATED_MEMORY_ID` exists in Postgres.
- Qdrant snapshots are readable and show collection/point totals.

If any step fails:

```bash
cd /home/trinm/Workspace/TVPL-AI-Workspace/mem0
$COMPOSE_CMD -f .ops/docker-compose.mem0-shared.yml --project-directory .ops logs mem0-api --tail=200
$COMPOSE_CMD -f .ops/docker-compose.mem0-shared.yml --project-directory .ops logs shared-services-ready --tail=120
```

---

### 3G) What to send me after Step 3

Please send:

1. Output of Step 3B (`ps`) showing service status.
2. Output headers from Step 3C (`/docs` and UI root).
3. Output of create + get-by-id + search from Step 3E.
4. Postgres before/after counts and row verification from Step 3F.
5. Qdrant before/after totals from Step 3F.
6. If any failure, include the exact error and `mem0-api` logs.
