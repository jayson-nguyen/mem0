Postgres: SERVER_IP:9045
Qdrant HTTP: SERVER_IP:9033
Qdrant gRPC: SERVER_IP:9034 (optional for many flows, but good to tunnel too)
Collect required access details before any config change
 We will need:
Server IP or DNS name.
SSH user (and bastion/jump host info if required).
Postgres credentials and database name for Mem0.
Qdrant auth requirement (API key or no auth).
Whether your machine is already on VPN.
Establish secure local tunnels (if using SSH tunnel path)
 Run this on your local machine:
 ssh -N
 -L 15432:127.0.0.1:9045
 -L 16333:127.0.0.1:9033
 -L 16334:127.0.0.1:9034
 YOUR_SSH_USER@YOUR_SERVER
If your company requires a bastion:
 ssh -J BASTION_USER@BASTION_HOST -N
 -L 15432:127.0.0.1:9045
 -L 16333:127.0.0.1:9033
 -L 16334:127.0.0.1:9034
 YOUR_SSH_USER@YOUR_SERVER
Verify connectivity before touching Mem0
 Check ports:
 nc -vz 127.0.0.1 15432
 nc -vz 127.0.0.1 16333
Check Postgres login:
 PGPASSWORD=YOUR_DB_PASSWORD psql -h 127.0.0.1 -p 15432 -U YOUR_DB_USER -d YOUR_DB_NAME -c "select now();"
Check Qdrant:
 curl -s http://127.0.0.1:16333/collections
Point Mem0 to these local forwarded endpoints
 Important detail:
If Mem0 runs on your host directly: use 127.0.0.1 and tunnel ports.
If Mem0 runs in Docker locally: container cannot use host 127.0.0.1 directly for your host tunnel.
For Dockerized Mem0, use host.docker.internal and add host-gateway mapping if needed.
Target values for Dockerized Mem0:
DATABASE_URL=postgresql://USER:PASSWORD@host.docker.internal:15432/DB_NAME
QDRANT_HOST=host.docker.internal
QDRANT_PORT=16333
Also ensure mem0-api service has:
extra_hosts entry mapping host.docker.internal to host-gateway (Linux).
Start Mem0 and run one write test
Start your Mem0 stack.
Verify API health/docs endpoint.
Create one test memory via UI or API call.
Read back memory via UI/API to confirm application-level success.
Validate data landed in both databases
 Postgres verification:
List tables and identify memory-related tables.
Check row counts before/after test.
Confirm one new record appears after your write.
Qdrant verification:
List collections.
Check collection point count increased after write.
Troubleshooting flow if anything fails
Connection refused: firewall/VPN/port allowlist issue.
Timeout: route or tunnel not established.
Auth failures: DB creds or Qdrant auth mismatch.
Mem0 container cannot reach tunnel: host.docker.internal mapping missing.
Data not persisted despite API success: check Mem0 logs for DB/Qdrant write errors.
Security note
 Your attached env file appears to contain a real NVIDIA API key. Rotate it after testing if this file was shared or committed anywhere.
