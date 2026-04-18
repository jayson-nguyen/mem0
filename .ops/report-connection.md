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