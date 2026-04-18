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
