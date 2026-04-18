# Report Step 1
## Company Server check
```
# 1) Confirm both containers are up and port mappings are present
CONTAINER ID   IMAGE                                              STATUS                  PORTS                                                                                                                                   NAMES
02423cc7331e   postgres:13-alpine                                 Up 14 hours             0.0.0.0:9045->5432/tcp, [::]:9045->5432/tcp                                                                                             postgres_share
be1dc6da2728   qdrant/qdrant:latest                               Up 14 hours             6335/tcp, 0.0.0.0:9033->6333/tcp, [::]:9033->6333/tcp, 0.0.0.0:9034->6334/tcp, [::]:9034->6334/tcp                                      qdrant

# 2) Confirm host is listening on expected published ports
LISTEN 0      4096         0.0.0.0:9033       0.0.0.0:*                                                
LISTEN 0      4096         0.0.0.0:9034       0.0.0.0:*                                                
LISTEN 0      4096         0.0.0.0:9045       0.0.0.0:*                                                
LISTEN 0      4096            [::]:9033          [::]:*                                                
LISTEN 0      4096            [::]:9034          [::]:*                                                
LISTEN 0      4096            [::]:9045          [::]:*

# 3) Quick HTTP health check for Qdrant from the server host
HTTP/1.1 200 OK
content-length: 112
vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
content-type: application/json
date: Fri, 17 Apr 2026 16:55:26 GMT

{"title":"qdrant - vector search engine","version":"1.17.1","commit":"eabee371fda447974a94d29fbaa675a6a596cc7b"}
```

## Local Machine check
```
# 1) Basic network reachability
PING 14.161.5.155 (14.161.5.155) 56(84) bytes of data.
64 bytes from 14.161.5.155: icmp_seq=1 ttl=49 time=7.30 ms
64 bytes from 14.161.5.155: icmp_seq=2 ttl=49 time=5.72 ms
64 bytes from 14.161.5.155: icmp_seq=3 ttl=49 time=6.64 ms

--- 14.161.5.155 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 5.717/6.554/7.301/0.649 ms

# 2) Test direct TCP access to published DB ports
(base) nmtri2110@Swuzz:~/Workspace/TVPL-AI/mem0$ nc -vz "$COMPANY_SERVER_HOST" 9045
Connection to 14.161.5.155 9045 port [tcp/*] succeeded!
(base) nmtri2110@Swuzz:~/Workspace/TVPL-AI/mem0$ nc -vz "$COMPANY_SERVER_HOST" 9033
Connection to 14.161.5.155 9033 port [tcp/*] succeeded!
(base) nmtri2110@Swuzz:~/Workspace/TVPL-AI/mem0$ nc -vz "$COMPANY_SERVER_HOST" 9034
Connection to 14.161.5.155 9034 port [tcp/*] succeeded!

# 3) Test Qrant HTTP endpoint directly
HTTP/1.1 200 OK
content-length: 112
content-type: application/json
vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
date: Fri, 17 Apr 2026 17:09:02 GMT

{"title":"qdrant - vector search engine","version":"1.17.1","commit":"eabee371fda447974a94d29fbaa675a6a596cc7b"}
```

# Report Step 2

## 2B) Update .ops/.env to remote DB endpoints (local machine)
```
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ export MEM0_DB_USER
export MEM0_DB_NAME
export MEM0_DB_PASSWORD

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ export ENCODED_DB_PASSWORD="$(python3 - <<'PY'
import os
from urllib.parse import quote
print(quote(os.environ['MEM0_DB_PASSWORD'], safe=''))
PY
)"

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ sed -i "s|^SHARED_POSTGRES_HOST=.*|SHARED_POSTGRES_HOST=${COMPANY_SERVER_HOST}|" .ops/.env

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ sed -i "s|^SHARED_POSTGRES_PORT=.*|SHARED_POSTGRES_PORT=${COMPANY_POSTGRES_PORT}|" .ops/.env

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ sed -i "s|^SHARED_QDRANT_HOST=.*|SHARED_QDRANT_HOST=${COMPANY_SERVER_HOST}|" .ops/.env

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ sed -i "s|^SHARED_QDRANT_PORT=.*|SHARED_QDRANT_PORT=${COMPANY_QDRANT_HTTP_PORT}|" .ops/.env

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ sed -i "s|^DATABASE_URL=.*|DATABASE_URL=postgresql://${MEM0_DB_USER}:${ENCODED_DB_PASSWORD}@${COMPANY_SERVER_HOST}:${COMPANY_POSTGRES_PORT}/${MEM0_DB_NAME}|" .ops/.env

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ sed -i "s|^QDRANT_HOST=.*|QDRANT_HOST=${COMPANY_SERVER_HOST}|" .ops/.env

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ sed -i "s|^QDRANT_PORT=.*|QDRANT_PORT=${COMPANY_QDRANT_HTTP_PORT}|" .ops/.env

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ grep -E '^(SHARED_POSTGRES_HOST|SHARED_POSTGRES_PORT|SHARED_QDRANT_HOST|SHARED_QDRANT_PORT|QDRANT_HOST|QDRANT_PORT)=' .ops/.env
SHARED_POSTGRES_HOST=14.161.5.155
SHARED_POSTGRES_PORT=9045
SHARED_QDRANT_HOST=14.161.5.155
SHARED_QDRANT_PORT=9033
QDRANT_HOST=14.161.5.155
QDRANT_PORT=9033

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ grep '^DATABASE_URL=' .ops/.env | sed -E 's#(postgresql://[^:]+:)[^@]+(@.*)#\1***\2#'
DATABASE_URL=postgresql://admin:***@14.161.5.155:9045/mem0_openmemory
```

## 2C) Validate remote DB reachability from Docker containers (local machine)
```
# Ensure external network declared in compose exists locally
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker network inspect shared_service_network >/dev/null 2>&1 || docker network create shared_service_network
87ba3e7615490e9ed77da5080e1156446aff18245ec8ecec75101bfd33ee99da

# 1) TCP checks from inside a container
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker run --rm busybox:1.36 sh -c "nc -zvw5 ${COMPANY_SERVER_HOST} ${COMPANY_POSTGRES_PORT} && nc -zvw5 ${COMPANY_SERVER_HOST} ${COMPANY_QDRANT_HTTP_PORT} && nc -zvw5 ${COMPANY_SERVER_HOST} ${COMPANY_QDRANT_GRPC_PORT}"
Unable to find image 'busybox:1.36' locally
1.36: Pulling from library/busybox
034d6572bf28: Pull complete 
5108bc06b83a: Download complete 
Digest: sha256:73aaf090f3d85aa34ee199857f03fa3a95c8ede2ffd4cc2cdb5b94e566b11662
Status: Downloaded newer image for busybox:1.36
14.161.5.155 (14.161.5.155:9045) open
14.161.5.155 (14.161.5.155:9033) open
14.161.5.155 (14.161.5.155:9034) open

# 2) Qdrant HTTP check from inside a container
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker run --rm curlimages/curl:8.7.1 -sS -i "http://${COMPANY_SERVER_HOST}:${COMPANY_QDRANT_HTTP_PORT}/" | sed -n '1,10p'
Unable to find image 'curlimages/curl:8.7.1' locally
8.7.1: Pulling from curlimages/curl
4ca545ee6d5d: Pull complete 
4abcf2066143: Pull complete 
1113023ab841: Pull complete 
Digest: sha256:25d29daeb9b14b89e2fa8cc17c70e4b188bca1466086907c2d9a4b56b59d8e21
Status: Downloaded newer image for curlimages/curl:8.7.1
HTTP/1.1 200 OK
content-length: 112
vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
content-type: application/json
date: Sat, 18 Apr 2026 02:48:05 GMT

{"title":"qdrant - vector search engine","version":"1.17.1","commit":"eabee371fda447974a94d29fbaa675a6a596cc7b"}(base)

# 3) Postgres authentication check from inside a container
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker run --rm -e PGPASSdocker run --rm -e PGPASSWORD="$MEM0_DB_PASSWORD" postgres:13-alpine \
   psql -h "$COMPANY_SERVER_HOST" -p "$COMPANY_POSTGRES_PORT" -U "$MEM0_DB_USER" -d "$MEM0_DB_NAME" -c "select now();"
Unable to find image 'postgres:13-alpine' locally
13-alpine: Pulling from library/postgres
2d35ebdb57d9: Pull complete 
191ad9697389: Pull complete 
a8d05d343dd2: Pull complete 
14981f3a93ca: Pull complete 
0e6c5d8eab6d: Pull complete 
e62b4937bf7f: Pull complete 
74c6de8827e1: Pull complete 
4f08f7c06c53: Pull complete 
fe7ec1f85969: Pull complete 
a64d9570aeee: Pull complete 
dbb6b142b9ce: Pull complete 
be6f4d070bb8: Download complete 
66872e5aa49c: Download complete 
Digest: sha256:fb9065b6e3e213bdc07edd372a5b2a26245840b7fb65d1fd8b6700106d51805c
Status: Downloaded newer image for postgres:13-alpine
psql: error: FATAL:  database "mem0_openmemory" does not exist
```

## 2E) Fix Postgres DB name mismatch (local machine)
```
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ grep '^DATABASE_URL=' .ops/.env | sed -E 's#(postgresql://[^:]+:)[^@]+(@[^/]+/)([^?]+).*#\1***\2\3#'
DATABASE_URL=postgresql://admin:***@14.161.5.155:9045/mem0_openmemory
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ export DATABASE_URL="$(grep '^DATABASE_URL=' .ops/.env | cut -d= -f2-)"
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ export POSTGRES_URL="${DATABASE_URL%/*}/postgres"
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker run --rm postgres:13-alpine \
   psql -w "$POSTGRES_URL" -Atc "select datname from pg_database where datistemplate=false order by 1;"
airflow
langfuse
postgres
tvpl_ai
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ read -rp "Use existing DB name: " MEM0_DB_NAME
Use existing DB name: mem0_openmemory
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ sed -i -E "s#^(DATABASE_URL=postgresql://[^/]+/)[^?]+#\1${MEM0_DB_NAME}#" .ops/.env
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ grep '^DATABASE_URL=' .ops/.env | sed -E 's#(postgresql://[^:]+:)[^@]+(@.*)#\1***\2#'
DATABASE_URL=postgresql://admin:***@14.161.5.155:9045/mem0_openmemory
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker run --rm postgres:13-alpine \
   psql -w "$POSTGRES_URL" -c "CREATE DATABASE mem0_openmemory;"
CREATE DATABASE
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ export DATABASE_URL="$(grep '^DATABASE_URL=' .ops/.env | cut -d= -f2-)"
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker run --rm postgres:13-alpine \
   psql -w "$DATABASE_URL" -c "select now();"
              now              
-------------------------------
 2026-04-18 11:08:20.682141+07
(1 row)
```

# Report Step 3

## 3B) Start Mem0 stack (local machine)
```
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker compose -f .ops/docker-compose.mem0-shared.yml --project-directory .ops up -d
[+] up 30/30
 ✔ Image mem0/openmemory-mcp:latest              Pulled                                                    99.1s
 ✔ Image mem0/openmemory-ui:latest               Pulled                                                    82.6s
 ✔ Network mem0-shared_default                   Created                                                    0.0s
 ✔ Container mem0-shared-shared-services-ready-1 Healthy                                                    7.4s
 ✘ Container mem0-shared-mem0-api-1              Error dependency mem0-api failed to start                 11.0s
 ✔ Container mem0-shared-mem0-ui-1               Created                                                    0.2s
dependency failed to start: container mem0-shared-mem0-api-1 is unhealthy
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED         STATUS                         PORTS     NAMES
b023eacf5e2d   mem0/openmemory-mcp:latest   "uvicorn main:app --…"   3 minutes ago   Restarting (1) 7 seconds ago             mem0-shared-mem0-api-1
f3dd15530e56   busybox:1.36                 "/bin/sh -ec 'until …"   3 minutes ago   Up 3 minutes (healthy)                   mem0-shared-shared-services-ready-1
```

## 3C) Health checks for API and UI (local machine)
```
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ curl -sS -i http://localhost:9100/docs | sed -n '1,10p'
curl: (7) Failed to connect to localhost port 9100 after 0 ms: Couldn't connect to server

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ curl -sS -i http://localhost:9101/ | sed -n '1,10p'
curl: (7) Failed to connect to localhost port 9101 after 0 ms: Couldn't connect to server

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ export MEM0_USER_ID="$(grep '^MEM0_USER_ID=' .ops/.env | cut -d= -f2-)"

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ curl -sS "http://localhost:9100/api/v1/stats/?user_id=${MEM0_USER_ID}"
curl: (7) Failed to connect to localhost port 9100 after 0 ms: Couldn't connect to server
```

## 3D) Capture baseline counts before write (local machine)
```
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ export DATABASE_URL="$(grep '^DATABASE_URL=' .ops/.env | cut -d= -f2-)"

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ export DATABASE_URL="$(grep '^DATABASE_URL=' .ops/.env | cut -d= -f2-)"

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ export QDRANT_PORT="$(grep '^QDRANT_PORT=' .ops/.env | cut -d= -f2-)"

(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ docker run --rm postgres:13-alpine psql -w "$DATABASE_URL" -Atc "select count(*) from memories;" | tee /tmp/mem0_step3_pg_before.txt
ERROR:  relation "memories" does not exist
LINE 1: select count(*) from memories;
                             ^
                             
(base) trinm@trinm-A6SBCCP:~/Workspace/TVPL-AI-Workspace/mem0$ python3 - <<'PY' | tee /tmp/mem0_step3_qdrant_before.txt
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
PYint(f"TOTAL_POINTS={total_points}")oints}\tvectors_count={vectors}")
Traceback (most recent call last):
  File "<stdin>", line 5, in <module>
  File "<frozen os>", line 717, in __getitem__
KeyError: 'QDRANT_HOST'
```