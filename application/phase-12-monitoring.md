# Phase 12 — Monitoring

## Goal

Monitor the production system so that failures can be detected before users report them.

Deployment is not complete just because the application is running.

A production system should answer:

- Is the application running?
- Is the container running?
- Is the API responding?
- Is HTTPS working?
- Is Nginx healthy?
- Is MongoDB reachable?
- Is the server running out of memory?
- Is disk space running low?
- Are errors increasing?
- Are containers restarting?
- Did the latest deployment actually become healthy?

The goal of this phase is to create a practical monitoring foundation without introducing unnecessary infrastructure.

---

# 1. Monitoring Model

The production system is:

    Internet
       |
       v
    HTTPS
       |
       v
    Nginx
       |
       v
    Docker
       |
       v
    Node.js
       |
       v
    MongoDB Atlas
       |
       +---- External APIs

Monitoring should exist at each important layer.

---

# 2. Four Important Monitoring Categories

A practical monitoring system can be divided into:

    Availability
    Performance
    Errors
    Resources

## Availability

Answers:

    "Is the service available?"

Examples:

- Health endpoint
- HTTPS availability
- Container status
- Nginx status
- MongoDB connectivity

## Performance

Answers:

    "Is the service becoming slow?"

Examples:

- Response time
- Request duration
- CPU usage
- Memory usage
- WebSocket connection behavior

## Errors

Answers:

    "Is something failing?"

Examples:

- HTTP 500 responses
- Authentication failures
- MongoDB errors
- External API failures
- Container crashes

## Resources

Answers:

    "Is the server running out of capacity?"

Examples:

- CPU
- RAM
- Disk
- Docker storage
- Network usage

---

# 3. Health Endpoint

The application already provides:

    GET /health

Example:

    curl.exe https://api.YOUR_DOMAIN/health

Expected response:

    {
      "status": "ok"
    }

A health endpoint should be simple and fast.

It should answer:

    "Can the API process a request?"

It should not perform expensive operations.

---

# 4. Health vs Readiness

These are related but different concepts.

## Liveness

Question:

    "Is the application process alive?"

Example:

    GET /health

## Readiness

Question:

    "Is the application ready to serve production traffic?"

A readiness check could eventually verify important dependencies such as:

    Application
       |
       +---- MongoDB
       +---- Required services

For the current deployment, the simple health endpoint provides the first monitoring layer.

A more advanced readiness endpoint can be added later if required.

---

# 5. Docker Monitoring

Check running containers:

    docker ps

Check Compose services:

    docker compose ps

Expected:

    support-assistant-backend
    Up

A stopped container should immediately be investigated.

---

# 6. Container Restart Policy

Production Compose uses:

    restart: unless-stopped

This means Docker can restart the container if the process exits unexpectedly.

Conceptually:

    Node.js crashes
          |
          v
    Container exits
          |
          v
    Docker detects exit
          |
          v
    Container restarts

This improves availability.

However:

> Restarting a container is not the same as fixing the underlying problem.

Repeated restarts may indicate:

- Application crash
- Database connection problem
- Missing environment variable
- Out-of-memory condition
- External service failure
- Code defect

---

# 7. Check Restart Count

Use:

    docker ps

or:

    docker inspect support-assistant-backend

Look for restart information.

A container that repeatedly restarts should be treated as an incident.

Example pattern:

    Start
      |
      v
    Crash
      |
      v
    Restart
      |
      v
    Crash
      |
      v
    Restart

This is a crash loop.

---

# 8. Container Logs

View recent logs:

    docker compose logs --tail=100

Follow logs:

    docker compose logs -f

Follow only the backend:

    docker compose logs -f backend

Logs are one of the first tools for production troubleshooting.

Typical questions:

    Did the application start?
    Did MongoDB connect?
    Did an exception occur?
    Did the server receive the request?
    Did an external API fail?

---

# 9. Log Retention

Logs should not grow indefinitely.

If Docker logs are never controlled, they can consume disk space.

The production system should eventually have a defined strategy for:

- Log size
- Rotation
- Retention
- Collection
- Searching
- Deletion

For the current small deployment, basic Docker and Nginx logs are sufficient as the starting point.

A centralized logging system can be introduced later.

---

# 10. Nginx Monitoring

Nginx has:

    Access logs
    Error logs

Typical locations:

    /var/log/nginx/access.log
    /var/log/nginx/error.log

Check recent access logs:

    sudo tail -n 100 /var/log/nginx/access.log

Check errors:

    sudo tail -n 100 /var/log/nginx/error.log

Follow errors:

    sudo tail -f /var/log/nginx/error.log

---

# 11. What Nginx Access Logs Tell Us

Access logs can help answer:

    Who requested the API?
    Which endpoint was requested?
    What HTTP status was returned?
    How frequently is the endpoint being called?

For example:

    200 -> successful request
    400 -> client-side validation/problem
    401 -> authentication failure
    403 -> forbidden
    404 -> resource not found
    500 -> server error
    502 -> upstream/backend problem
    504 -> upstream timeout

A sudden increase in:

    500
    502
    504

should be investigated.

---

# 12. Nginx 502

One particularly useful production error is:

    502 Bad Gateway

With the current architecture:

    Client
       |
       v
    Nginx
       |
       X
    Node.js unavailable

Nginx may return:

    502 Bad Gateway

Possible causes:

- Container stopped
- Node.js process crashed
- Port 3000 unavailable
- Application startup failure
- Docker networking problem

Troubleshooting:

    docker compose ps

then:

    docker compose logs --tail=100

Then test locally from EC2:

    curl http://127.0.0.1:3000/health

---

# 13. HTTPS Monitoring

Production health should be checked through the public URL.

Example:

    curl.exe https://api.YOUR_DOMAIN/health

This verifies more than the application.

It tests:

    DNS
      |
      v
    Internet
      |
      v
    HTTPS
      |
      v
    Nginx
      |
      v
    Node.js

Therefore a public health check is more useful than checking only:

    localhost:3000

---

# 14. Internal vs External Health Check

There are two useful checks.

## Internal

    curl http://127.0.0.1:3000/health

Tests:

    Docker -> Node.js

## External

    curl https://api.YOUR_DOMAIN/health

Tests:

    Internet -> DNS -> HTTPS -> Nginx -> Node.js

If internal works but external fails:

    Node.js is probably working.

Investigate:

    Nginx
    DNS
    Security Group
    HTTPS
    Network

---

# 15. Deployment Health Verification

A deployment should not be considered successful only because:

    docker compose up -d

completed.

The application must be verified.

Conceptually:

    Deploy
      |
      v
    Container starts
      |
      v
    Container healthy
      |
      v
    API responds
      |
      v
    HTTPS responds
      |
      v
    Deployment accepted

This is especially important for CI/CD.

---

# 16. Current CI/CD Limitation

The current deployment pipeline performs:

    docker compose pull
    docker compose up -d
    docker compose ps

The next improvement is to add an application health check.

Conceptually:

    docker compose up -d
           |
           v
    Wait for startup
           |
           v
    curl health endpoint
           |
      +----+----+
      |         |
     OK       FAIL
      |         |
      v         v
   Success    Rollback

This connects monitoring with deployment safety.

---

# 17. Health Check in CI/CD

A future deployment step can perform:

    curl https://api.YOUR_DOMAIN/health

Then inspect the HTTP response.

Expected:

    HTTP 200

If the endpoint does not respond successfully:

    Deployment should fail.

The next phase after monitoring can improve this into automatic rollback.

---

# 18. CPU Monitoring

EC2 CPU usage should be monitored.

Basic command:

    top

Alternative:

    htop

if installed.

Docker-specific monitoring:

    docker stats

Example:

    docker stats

This can show:

- CPU
- Memory
- Network
- Block I/O
- Process information

---

# 19. Docker Resource Monitoring

Run:

    docker stats

This provides a quick view of container resource consumption.

Important metrics:

    CPU %
    MEM USAGE
    MEM %
    NET I/O
    BLOCK I/O

Example interpretation:

    CPU suddenly high
        |
        v
    Investigate workload

    Memory continuously increasing
        |
        v
    Investigate memory leak

    Disk I/O unexpectedly high
        |
        v
    Investigate database/logging/workload

---

# 20. Memory Monitoring

The current EC2 instance is intentionally small.

Memory therefore needs attention.

Check:

    free -h

Example categories:

    total
    used
    free
    available
    swap

High memory pressure can cause:

- Slow application
- Container crashes
- OOM kills
- SSH problems
- System instability

---

# 21. OOM Condition

OOM means:

    Out Of Memory

Conceptually:

    Memory usage increases
          |
          v
    Available RAM decreases
          |
          v
    System reaches memory pressure
          |
          v
    Process/container may be killed

If the Node.js container repeatedly restarts without an obvious application error, memory pressure should be investigated.

Useful command:

    dmesg | grep -i "out of memory"

Depending on system permissions and configuration, logs may also be available through:

    journalctl

---

# 22. Disk Monitoring

Check disk:

    df -h

Important locations include:

    /
    /var
    Docker storage
    Nginx logs

A full disk can cause:

- Docker failures
- Log failures
- Database-related issues
- Application failures
- SSH problems

Disk monitoring is therefore critical even for a small server.

---

# 23. Docker Disk Usage

Check Docker storage:

    docker system df

This shows Docker resource usage.

Over time, unused:

- Images
- Containers
- Volumes
- Build cache

can consume disk space.

Do not blindly run:

    docker system prune -a

on production.

First understand what is being used.

---

# 24. Why Blind Docker Cleanup Is Dangerous

Commands such as:

    docker system prune -a

can remove resources that may be needed later.

Production cleanup should be deliberate.

First inspect:

    docker system df

Then determine:

    What is unused?
    What is required for rollback?
    What images are retained?
    What volumes are important?

Only then perform cleanup.

---

# 25. MongoDB Monitoring

MongoDB is hosted by MongoDB Atlas.

Application-level monitoring should detect:

- Connection failures
- Query errors
- Authentication failures
- Timeout errors
- Slow operations

Atlas should separately be used for database-level monitoring and operational visibility.

The application should not assume:

    MongoDB is always available.

External dependencies can fail.

---

# 26. External API Monitoring

The application depends on external services.

Examples:

    AssemblyAI
    Groq
    Anthropic
    Other APIs

External APIs can fail because of:

- Timeout
- Rate limit
- Authentication failure
- Service outage
- Invalid request
- Network problem

The application should log these failures safely.

Do not log the API key.

---

# 27. Rate Limit Monitoring

External services may return rate-limit responses.

For example:

    HTTP 429

This means the application should distinguish:

    Application bug

from:

    External provider rate limit

The response should be logged with safe metadata such as:

    provider
    endpoint/category
    status code
    request context
    timestamp

but not:

    API key
    authorization header

---

# 28. Application Error Monitoring

Application errors should be distinguishable from expected client errors.

Examples:

    400
    401
    403
    404

are not necessarily application failures.

Unexpected:

    500

requires more attention.

Monitoring should eventually track:

    total requests
    successful requests
    client errors
    server errors
    response latency

---

# 29. Error Rate

One useful production metric is:

    Error Rate

Conceptually:

    Error Rate =
    Failed Requests / Total Requests

For example:

    10 failed
    1,000 total

means:

    1% error rate

The exact acceptable threshold depends on the application.

Do not blindly use a universal threshold.

---

# 30. Response Time

Another important metric is:

    Request Latency

Example:

    /health
       50 ms

    /login
       120 ms

    /conversation
       300 ms

A sudden increase can indicate:

- Database slowdown
- CPU pressure
- External API latency
- Network problems
- Application bottleneck

Latency should be monitored over time rather than from one request.

---

# 31. WebSocket Monitoring

The application uses WebSockets for streaming.

Important metrics include:

    Active connections
    Connection failures
    Connection duration
    Disconnect rate
    Authentication failures
    External streaming failures

The application already maintains active connection information internally.

Future monitoring can expose safe metrics without exposing user data.

---

# 32. WebSocket Connection Failure

Conceptually:

    Electron Client
          |
          | WSS
          v
       Nginx
          |
          v
       Node.js
          |
          v
      Streaming API

A failure at any layer can terminate the connection.

Troubleshooting should therefore check:

    1. DNS
    2. HTTPS
    3. Nginx
    4. Docker
    5. Node.js
    6. External streaming service

---

# 33. Monitoring Without Kubernetes

This deployment intentionally does not require Kubernetes.

For the current architecture:

    One EC2
       |
       +---- Nginx
       +---- Docker Compose
       +---- Node.js

basic monitoring can be implemented using:

- Health endpoint
- Docker status
- Docker logs
- Nginx logs
- Linux metrics
- External uptime checks
- Cloud/provider monitoring

A larger architecture may eventually justify more advanced orchestration and observability.

---

# 34. Uptime Monitoring

An external uptime monitor can periodically request:

    https://api.YOUR_DOMAIN/health

Conceptually:

    Monitoring Service
           |
           | HTTPS
           v
    api.YOUR_DOMAIN
           |
           v
         Nginx
           |
           v
        Node.js

If the service becomes unavailable, an alert can be generated.

The important principle is that the monitor should be external to the server being monitored.

Otherwise:

    EC2 is down
       |
       v
    Monitor on EC2 is also down

and no alert is generated.

---

# 35. Alerting

Monitoring is useful only if important failures result in action.

Possible alerts:

    API unavailable
    Health check failed
    High error rate
    High CPU
    High memory
    Low disk space
    Container repeatedly restarting
    SSL certificate approaching expiry

Alerts should avoid creating noise.

Not every warning requires an immediate page/notification.

---

# 36. SSL Certificate Monitoring

HTTPS uses a TLS certificate.

Certificates have an expiration date.

The system should verify:

    Certificate valid
    Certificate not close to expiry
    Automatic renewal functioning

The current setup uses Certbot and automatic renewal.

Do not assume renewal works forever.

Periodically verify:

    sudo certbot renew --dry-run

A successful dry run indicates that the renewal process can execute successfully.

---

# 37. Systemd Monitoring

Linux services such as Nginx can be checked with:

    sudo systemctl status nginx

Check Docker:

    sudo systemctl status docker

These commands help distinguish:

    Nginx problem

from:

    Docker problem

from:

    Application problem

---

# 38. Basic Production Troubleshooting Flow

When the API is unavailable:

    Step 1
    Check DNS

        nslookup api.YOUR_DOMAIN

    Step 2
    Check HTTPS

        curl.exe https://api.YOUR_DOMAIN/health

    Step 3
    Check Nginx

        sudo systemctl status nginx

    Step 4
    Check Docker

        docker compose ps

    Step 5
    Check application logs

        docker compose logs --tail=100

    Step 6
    Check local application

        curl http://127.0.0.1:3000/health

    Step 7
    Check resources

        free -h
        df -h
        docker stats

This provides a bottom-up troubleshooting path.

---

# 39. Troubleshooting Decision Tree

```text
                 API unavailable
                       |
                       v
                DNS resolving?
                  /         \
                NO           YES
                |             |
             DNS issue        v
                         HTTPS working?
                           /       \
                         NO         YES
                         |           |
                    Nginx/TLS       v
                              Container running?
                                /       \
                              NO         YES
                              |           |
                         Docker/app      v
                              issue   Local health?
                                       /      \
                                     NO        YES
                                     |          |
                              Application     Nginx/
                              issue           network



40. Monitoring Philosophy

When troubleshooting production:

Measure first. Change second.

Do not immediately:

restart everything
delete containers
prune Docker
change DNS
change Security Groups
modify Nginx

First gather evidence.

For example:

docker compose ps
docker compose logs --tail=100
curl http://127.0.0.1:3000/health
curl https://api.YOUR_DOMAIN/health
df -h
free -h

Then make the smallest required change.

41. Deployment Monitoring

A production deployment should have this lifecycle:

Developer
    |
    v
Git Push
    |
    v
GitLab CI
    |
    v
Docker Build
    |
    v
Container Registry
    |
    v
Manual Production Deploy
    |
    v
Docker Compose
    |
    v
Health Check
    |
    +---- PASS ----> Deployment successful
    |
    +---- FAIL ----> Investigate / Rollback

Monitoring therefore becomes part of deployment safety.

42. Current Monitoring Commands

Useful commands for this project:

Application
curl http://127.0.0.1:3000/health
Public API
curl.exe https://api.YOUR_DOMAIN/health
Containers
docker compose ps
Logs
docker compose logs --tail=100
Live logs
docker compose logs -f
Resource usage
docker stats
Disk
df -h
Memory
free -h
Docker storage
docker system df
Nginx
sudo systemctl status nginx
Nginx errors
sudo tail -n 100 /var/log/nginx/error.log
Docker service
sudo systemctl status docker
Certificate renewal test
sudo certbot renew --dry-run
43. What Should Be Automated

Manual commands are useful during troubleshooting.

But recurring checks should eventually be automated.

Potential automated checks:

Health check
SSL expiry
Disk usage
Container status
Error rate
CPU
Memory
Uptime

The purpose of automation is:

Detect
   |
   v
Alert
   |
   v
Investigate
   |
   v
Recover
44. Monitoring Maturity

The current system starts with basic operational monitoring.

Level 1 — Basic
Health endpoint
Docker status
Logs
Linux metrics
HTTPS monitoring
Level 2 — Automated
Uptime checks
Alerts
SSL monitoring
Disk alerts
Restart detection
Level 3 — Observability
Metrics
Centralized logs
Distributed tracing
Dashboards
Request correlation
Detailed performance analysis

The project currently focuses on Level 1 and the foundation for Level 2.

45. Avoid Overengineering

The current deployment is:

One EC2
One backend container
Nginx
MongoDB Atlas

It does not need a large observability platform just to monitor whether the API is alive.

Start with:

Health
Logs
CPU
Memory
Disk
Uptime
Alerts

Introduce more infrastructure only when the system requires it.

46. Production Incident Example

Suppose users report:

"The application is not connecting."

First check:

curl.exe https://api.YOUR_DOMAIN/health

If it fails, connect to EC2.

Check:

docker compose ps

If container is down:

docker compose logs --tail=100

If container is running:

curl http://127.0.0.1:3000/health

If internal health works but public HTTPS fails:

sudo systemctl status nginx

Then inspect:

sudo tail -n 100 /var/log/nginx/error.log

This is much safer than immediately redeploying.

47. Production Incident Example — High Memory

Suppose:

docker stats

shows continuously increasing memory usage.

Investigation:

1. Confirm memory growth.
2. Check application logs.
3. Check active connections.
4. Check request patterns.
5. Check external API behavior.
6. Determine whether the process is leaking memory.
7. Fix the application.
8. Deploy a new image.
9. Monitor again.

Do not treat:

restart container

as the permanent fix.

Restarting only hides the symptom if a memory leak remains.

48. Production Incident Example — Disk Full

Suppose:

df -h

shows the root filesystem nearly full.

Investigate:

docker system df

and:

sudo du -sh /var/log/*

Determine what is consuming space.

Possible causes:

Docker images
Docker logs
Nginx logs
Temporary files
Other application data

Clean only resources that are confirmed safe to remove.

Then introduce retention/rotation so the problem does not repeat.

49. Monitoring and Rollback

Monitoring and rollback are connected.

Example:

Deploy Version A
       |
       v
Health OK
       |
       v
Traffic
       |
       v
Error rate increases
       |
       v
Investigate
       |
       v
Rollback to Version B
       |
       v
Health OK

The ability to detect a problem is only half of the recovery process.

The other half is having a known-good version available.

That is why commit-SHA image tagging was established earlier.

50. Monitoring Security

Monitoring data can contain sensitive information.

Protect:

Logs
Metrics
Error traces
User identifiers
Request metadata
Internal hostnames
Infrastructure information

Do not make internal monitoring dashboards publicly accessible without authentication.

Monitoring infrastructure can become an attack target itself.

51. Current Monitoring Status

Completed foundation:

Health endpoint
Docker monitoring commands
Container logs
Nginx logs
Linux resource checks
HTTPS health verification
MongoDB dependency awareness
External API monitoring considerations
WebSocket monitoring considerations
Certificate renewal verification
Production troubleshooting flow

Remaining:

Automated external uptime monitoring
Alerting
Automated deployment health check
Automatic rollback after failed health check
Centralized logging if required
Metrics dashboard if required
Resource alerts
Formal incident-response process
52. Final Production Monitoring Checklist
Availability
 /health endpoint
 HTTPS health verification
 Docker status check
 Nginx status check
 External uptime monitoring
Logs
 Docker logs
 Nginx access logs
 Nginx error logs
 Log rotation strategy
 Centralized logging if required
Resources
 CPU monitoring
 Memory monitoring
 Disk monitoring
 Docker storage monitoring
 Automated resource alerts
Application
 Health endpoint
 Error logging
 External API awareness
 WebSocket monitoring considerations
 Request latency metrics
 Error-rate metrics
Deployment
 Commit-SHA image
 Manual production deployment
 Container verification
 Automated health check
 Automatic rollback
Infrastructure
 Nginx
 HTTPS
 Docker
 EC2
 MongoDB Atlas
 Backup/recovery validation
53. Final Architecture
                         Internet
                            |
                            v
                    +---------------+
                    | DNS / Domain  |
                    +---------------+
                            |
                            v
                    +---------------+
                    | HTTPS / WSS   |
                    +---------------+
                            |
                            v
                    +---------------+
                    |     Nginx     |
                    | Reverse Proxy |
                    +---------------+
                            |
                            v
                    +---------------+
                    |    Docker     |
                    |   Node.js API |
                    +---------------+
                       |          |
                       |          |
                       v          v
                MongoDB Atlas   External APIs
                       |
                       |
                       v
                  Application Data


Monitoring:

        External Uptime Monitor
                  |
                  v
             /health
                  |
                  v
                Nginx
                  |
                  v
              Node.js

        EC2 Monitoring
              |
       +------+------+------+
       |      |      |      |
      CPU    RAM   Disk   Docker
54. Complete Deployment Lifecycle

At this point, the documented deployment journey is:

Local Development
       |
       v
Git
       |
       v
Docker
       |
       v
GitLab Container Registry
       |
       v
GitLab CI/CD
       |
       v
AWS EC2
       |
       v
Nginx
       |
       v
Domain
       |
       v
HTTPS
       |
       v
Production Deployment
       |
       v
Rollback Strategy
       |
       v
Security
       |
       v
Monitoring

This represents the complete engineering journey documented by this repository.

55. Project Completion Status

The portfolio now documents:

Phase 01 — Local Development
Phase 02 — Git
Phase 03 — Docker
Phase 04 — Container Registry
Phase 05 — GitLab CI/CD
Phase 06 — AWS EC2
Phase 07 — Nginx
Phase 08 — Domain + HTTPS
Phase 09 — Production Deployment
Phase 10 — Rollback
Phase 11 — Security
Phase 12 — Monitoring

The major deployment lifecycle is now documented.

56. Remaining Real-World Work

The documentation is complete at the baseline architecture level.

The real production system still has improvement opportunities:

Complete credential rotation after the security exposure.
Remove public access to port 3000.
Harden SSH.
Improve CI SSH key handling.
Add proper automated tests to CI.
Add automated deployment health checks.
Add automatic rollback.
Define DEV / TEST / PROD environment strategy.
Implement monitoring and alerts.
Validate MongoDB backup/recovery.
Deploy the public website separately.
Define the Electron application's release/distribution process.

These are implementation improvements rather than prerequisites for understanding the deployment architecture.

57. Final Lessons
Lesson 1

A production deployment is not just:

docker run

It is a complete system:

Code
Git
Docker
Registry
CI/CD
Infrastructure
Reverse Proxy
DNS
HTTPS
Deployment
Rollback
Security
Monitoring
Lesson 2

Build once and deploy the same immutable artifact.

Lesson 3

Use commit SHA tags instead of relying on mutable latest for production releases.

Lesson 4

Keep secrets outside source code and Docker images.

Lesson 5

Expose only the required network ports.

Lesson 6

Monitoring should tell you when something is wrong before users have to tell you.

Lesson 7

Rollback must be based on a known-good artifact.

Lesson 8

Do not overengineer infrastructure before the workload requires it.

Lesson 9

Security, deployment, and monitoring are connected.

Lesson 10

Production engineering is a continuous process, not a one-time deployment.