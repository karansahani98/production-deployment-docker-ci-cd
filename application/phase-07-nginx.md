# Phase 07 — Nginx

## 1. Goal

The goal of this phase is to document how Nginx was introduced as the reverse proxy in front of the Dockerized Node.js backend.

Before Nginx, the backend was directly accessible through the application port:

```text
Internet
   |
   v
EC2
   |
   v
Docker :3000
   |
   v
Node.js Backend
```

After Nginx:

```text
Internet
   |
   v
Nginx
   |
   v
127.0.0.1:3000
   |
   v
Docker Container
   |
   v
Node.js Backend
```

Nginx becomes the public entry point for the API.

---

# 2. Why Nginx?

The Node.js application listens on:

```text
3000
```

That is an application port.

It should not be responsible for handling every public-facing concern.

Nginx provides a dedicated reverse-proxy layer.

The architecture becomes:

```text
Client
   |
   v
Nginx
   |
   v
Node.js
```

Nginx can handle:

- HTTP requests
- HTTPS termination
- Domain-based routing
- Reverse proxying
- WebSocket forwarding
- Request headers
- Connection management

This gives a clean separation:

```text
Nginx
   |
   +--> Public traffic
   |
   +--> TLS
   |
   +--> Routing
   |
   v
Node.js
   |
   +--> Application logic
```

---

# 3. Reverse Proxy Concept

A reverse proxy receives requests from clients and forwards them to an internal application.

For example:

```text
Client
   |
   | https://api.example.com/users
   v
Nginx
   |
   | http://127.0.0.1:3000/users
   v
Node.js
```

The client does not need to know that Node.js is running on port `3000`.

The public interface is:

```text
https://api.example.com
```

while the internal application is:

```text
http://127.0.0.1:3000
```

---

# 4. Why Not Expose Node.js Directly?

A simple deployment could expose:

```text
http://YOUR_EC2_IP:3000
```

However, this creates an architecture where the application itself becomes the public network endpoint.

Using Nginx provides:

```text
Internet
   |
   v
Nginx :80 / :443
   |
   v
Node.js :3000
```

The application port can therefore remain internal.

This also makes it easier to introduce:

```text
HTTPS
Domain routing
WebSocket proxying
Security controls
Future services
```

---

# 5. Nginx Installation

Nginx was installed on the Ubuntu EC2 server.

Typical installation:

```bash
sudo apt update
sudo apt install nginx -y
```

After installation, verify:

```bash
nginx -v
```

Then check the service:

```bash
sudo systemctl status nginx
```

The expected state is:

```text
active (running)
```

---

# 6. Enable Nginx at Boot

Nginx should start automatically after the server restarts.

Verify:

```bash
sudo systemctl is-enabled nginx
```

If required:

```bash
sudo systemctl enable nginx
```

The desired infrastructure behavior is:

```text
EC2 reboot
    |
    v
Docker
    |
    v
Backend container

EC2 reboot
    |
    v
Nginx
    |
    v
Reverse proxy available
```

---

# 7. Nginx Directory Structure

Ubuntu commonly stores Nginx site configurations under:

```text
/etc/nginx/
```

Important directories include:

```text
/etc/nginx/
    |
    +--> sites-available/
    |
    +--> sites-enabled/
```

The general approach is:

```text
sites-available
      |
      | configuration
      v
sites-enabled
      |
      | symbolic link
      v
Active Nginx site
```

This allows configurations to be enabled or disabled cleanly.

---

# 8. Application Nginx Configuration

A dedicated Nginx configuration was created for the backend.

Example:

```text
/etc/nginx/sites-available/support-assistant
```

For public documentation, the actual application name can be replaced with:

```text
YOUR_APPLICATION
```

---

# 9. Basic Reverse Proxy Configuration

The configuration follows this pattern:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

The actual production domain is intentionally not included in this public repository.

Use:

```text
api.example.com
```

as the public example.

---

# 10. Understanding server_name

The line:

```nginx
server_name api.example.com;
```

tells Nginx which hostname this server block should handle.

The request:

```text
http://api.example.com
```

is therefore routed to this configuration.

Conceptually:

```text
api.example.com
       |
       v
Nginx server block
       |
       v
127.0.0.1:3000
```

---

# 11. Understanding proxy_pass

The key line is:

```nginx
proxy_pass http://127.0.0.1:3000;
```

This means:

```text
Public request
      |
      v
Nginx
      |
      | forward request
      v
127.0.0.1:3000
```

The Node.js application continues to listen on port `3000`.

Nginx simply forwards the request.

---

# 12. Proxy Headers

The configuration includes:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

These headers preserve useful request information when Nginx forwards the request to Node.js.

Conceptually:

```text
Client
   |
   | original request information
   v
Nginx
   |
   | forwarded headers
   v
Node.js
```

This is useful when the application needs information about the original request.

---

# 13. WebSocket Support

The backend also uses WebSockets.

This means normal HTTP reverse proxy configuration is not enough.

The Nginx configuration includes:

```nginx
proxy_http_version 1.1;

proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

These headers allow the WebSocket upgrade process.

The architecture becomes:

```text
Electron / Client
       |
       | WebSocket
       v
     Nginx
       |
       | Upgrade
       v
Node.js WebSocket Server
```

This was important because the backend contains a WebSocket endpoint.

---

# 14. Why WebSocket Configuration Matters

A normal HTTP request looks like:

```text
Client
   |
   | HTTP request
   v
Nginx
   |
   v
Node.js
```

A WebSocket connection begins as an HTTP request and then upgrades the connection.

Conceptually:

```text
Client
   |
   | HTTP Upgrade
   v
Nginx
   |
   | Upgrade
   v
WebSocket
   |
   v
Node.js
```

Without the required upgrade headers, WebSocket connections may fail even though normal API requests work.

---

# 15. Creating the Site Configuration

Create:

```bash
sudo nano /etc/nginx/sites-available/YOUR_APPLICATION
```

Then add the appropriate reverse-proxy configuration.

Example:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

# 16. Enabling the Site

Create a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/YOUR_APPLICATION \
/etc/nginx/sites-enabled/YOUR_APPLICATION
```

This enables the configuration.

The structure becomes:

```text
sites-available
       |
       +--> YOUR_APPLICATION
       |
       v
sites-enabled
       |
       +--> YOUR_APPLICATION
```

---

# 17. Default Nginx Site

Ubuntu may have a default Nginx configuration:

```text
/etc/nginx/sites-enabled/default
```

The default site was removed so that the intended application configuration handled the request.

Example:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

This should only be done when the replacement configuration is ready.

---

# 18. Validate Nginx Configuration

Never reload Nginx blindly after changing its configuration.

First run:

```bash
sudo nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

Only after a successful configuration test should Nginx be reloaded.

```bash
sudo systemctl reload nginx
```

This is an important production habit.

---

# 19. Why Reload Instead of Restart?

For configuration changes:

```bash
sudo systemctl reload nginx
```

is generally preferable to:

```bash
sudo systemctl restart nginx
```

A reload allows Nginx to apply configuration changes while minimizing disruption to existing connections.

The workflow is:

```text
Edit configuration
       |
       v
nginx -t
       |
       | success
       v
systemctl reload nginx
```

---

# 20. Testing HTTP

After Nginx is configured, test the API through the domain.

Example:

```bash
curl http://api.example.com/health
```

Expected:

```json
{
  "status": "ok"
}
```

This verifies:

```text
Client
   |
   v
DNS
   |
   v
Nginx
   |
   v
Docker
   |
   v
Node.js
```

---

# 21. Testing Nginx Locally

Before debugging DNS, it can be useful to test the backend directly.

```bash
curl http://127.0.0.1:3000/health
```

If this works:

```text
Node.js + Docker = working
```

Then test:

```bash
curl http://api.example.com/health
```

If this works:

```text
Node.js + Docker + Nginx + DNS = working
```

This layered approach makes troubleshooting easier.

---

# 22. DNS Relationship

The API domain must resolve to the EC2 public/Elastic IP.

Conceptually:

```text
api.example.com
       |
       | DNS A record
       v
YOUR_EC2_PUBLIC_IP
       |
       v
EC2
       |
       v
Nginx
```

The actual production domain and IP are intentionally excluded from this public repository.

Use:

```text
YOUR_API_DOMAIN
YOUR_EC2_PUBLIC_IP
```

in documentation.

---

# 23. DNS Verification

From a client machine:

```powershell
nslookup api.example.com
```

The response should resolve to the expected public IP.

Example:

```text
Name:
api.example.com

Address:
YOUR_EC2_PUBLIC_IP
```

This confirms that DNS is pointing toward the intended server.

---

# 24. Request Flow

At this stage, an API request follows:

```text
Browser / Electron
       |
       | HTTP
       v
api.example.com
       |
       v
DNS
       |
       v
EC2
       |
       v
Nginx :80
       |
       | proxy_pass
       v
127.0.0.1:3000
       |
       v
Docker Container
       |
       v
Node.js Backend
```

For WebSocket:

```text
Electron
    |
    | WebSocket
    v
api.example.com
    |
    v
Nginx
    |
    | Upgrade
    v
Node.js WebSocket
```

---

# 25. Nginx and Docker Relationship

Nginx runs directly on the EC2 host.

The Node.js backend runs inside Docker.

Therefore:

```text
EC2 Host
   |
   +--> Nginx
   |
   +--> Docker
          |
          +--> Node.js Container
```

Nginx forwards requests to:

```text
127.0.0.1:3000
```

which is mapped to the backend container.

---

# 26. Why Nginx Runs Outside Docker

Nginx could also be containerized.

However, for this initial deployment, Nginx was installed directly on the EC2 host.

The resulting infrastructure is simple:

```text
Host
 |
 +--> Nginx
 |
 +--> Docker
       |
       +--> Backend
```

This keeps the public reverse-proxy layer independent from the application container.

It also makes initial HTTPS configuration straightforward.

A future architecture could containerize Nginx if there is a reason to standardize the entire infrastructure as containers.

---

# 27. HTTP Before HTTPS

The initial Nginx configuration listens on:

```text
80
```

which is HTTP.

The initial flow is:

```text
HTTP
 |
 v
Nginx
 |
 v
Backend
```

Once the domain is working correctly, HTTPS can be introduced.

The target architecture becomes:

```text
HTTPS :443
     |
     v
   Nginx
     |
     v
Backend :3000
```

HTTPS and certificate management are documented in Phase 08.

---

# 28. Security Group Relationship

The AWS Security Group must allow the traffic required by Nginx.

The public web traffic is:

```text
HTTP  :80
HTTPS :443
```

SSH is used for administration.

The backend port:

```text
3000
```

should eventually be restricted from public access because Nginx is the public entry point.

The exact Security Group rules are intentionally not included in the public repository.

---

# 29. Problem: Nginx Configuration Errors

If:

```bash
sudo nginx -t
```

fails, do not reload Nginx.

First inspect the error.

Common causes include:

```text
Syntax error
Missing semicolon
Incorrect server block
Invalid directive
Wrong file path
Duplicate configuration
```

The correct workflow is:

```text
Edit
  |
  v
nginx -t
  |
  +--> Failed --> Fix configuration
  |
  +--> Success
          |
          v
       Reload
```

---

# 30. Problem: API Does Not Respond

If:

```bash
curl http://api.example.com/health
```

fails, debug from the bottom upward.

### Step 1 — Check container

```bash
docker ps
```

### Step 2 — Check application logs

```bash
docker compose logs backend
```

### Step 3 — Test Node.js directly

```bash
curl http://127.0.0.1:3000/health
```

### Step 4 — Check Nginx

```bash
sudo nginx -t
```

### Step 5 — Check Nginx service

```bash
sudo systemctl status nginx
```

### Step 6 — Check DNS

```bash
nslookup api.example.com
```

This identifies which layer is failing.

---

# 31. Problem: WebSocket Does Not Connect

If normal API requests work but WebSocket does not:

Check that the Nginx configuration contains:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

Then test the configuration:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

Then inspect backend logs:

```bash
docker compose logs -f backend
```

The important distinction is:

```text
HTTP works
+
WebSocket fails
=
Check proxy upgrade configuration
```

---

# 32. Nginx Logs

Nginx provides access and error logs.

Typical locations:

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

View recent errors:

```bash
sudo tail -f /var/log/nginx/error.log
```

View requests:

```bash
sudo tail -f /var/log/nginx/access.log
```

These logs are useful when the request reaches Nginx but does not reach the backend correctly.

---

# 33. Layered Troubleshooting Model

A useful production debugging model is:

```text
Layer 1
EC2
 |
 v
Layer 2
Docker
 |
 v
Layer 3
Node.js
 |
 v
Layer 4
Nginx
 |
 v
Layer 5
DNS
 |
 v
Layer 6
HTTPS
```

Test each layer separately.

For example:

```text
127.0.0.1:3000
      |
      v
Node.js
```

then:

```text
api.example.com
      |
      v
Nginx
```

then:

```text
https://api.example.com
      |
      v
HTTPS + Nginx
```

This prevents debugging multiple unknowns simultaneously.

---

# 34. Important Nginx Configuration

The generalized production configuration is:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

This configuration is intentionally generalized.

Do not replace the public example with real production credentials, IP addresses, or private infrastructure details.

---

# 35. Important Decisions

## Decision 1 — Use Nginx as reverse proxy

Chosen:

```text
Internet
   |
   v
Nginx
   |
   v
Node.js
```

Reason:

- Public entry point
- Reverse proxy
- HTTPS termination
- Domain routing
- WebSocket support
- Separation of infrastructure and application

---

## Decision 2 — Keep Node.js on port 3000

The application continues listening on:

```text
3000
```

Nginx handles public traffic.

This avoids unnecessary application changes.

---

## Decision 3 — Support WebSockets

The backend uses WebSockets, so the Nginx configuration explicitly supports HTTP upgrade requests.

---

## Decision 4 — Validate before reload

Every configuration change follows:

```bash
sudo nginx -t
```

before:

```bash
sudo systemctl reload nginx
```

---

## Decision 5 — Domain-based routing

The API is accessed through a hostname instead of exposing the application directly through an IP and port.

---

# 36. Lessons Learned

### Lesson 1

Nginx should be treated as the public entry point while Node.js remains the application layer.

### Lesson 2

A working Docker container does not guarantee a working public API.

The request must pass through:

```text
DNS
Nginx
Docker
Node.js
```

### Lesson 3

WebSockets require explicit proxy upgrade configuration.

### Lesson 4

Always run:

```bash
nginx -t
```

before reloading Nginx.

### Lesson 5

Troubleshoot infrastructure layer by layer.

### Lesson 6

The backend application port does not need to be the public API endpoint.

### Lesson 7

Nginx provides a clean place to terminate HTTPS without changing the Node.js application itself.

---

# 37. Verification Checklist

Before considering this phase complete:

- [x] Nginx installed
- [x] Nginx service running
- [x] Nginx enabled at boot
- [x] Application server block created
- [x] Reverse proxy configured
- [x] `proxy_pass` configured
- [x] Proxy headers configured
- [x] WebSocket upgrade headers configured
- [x] Site enabled
- [x] Default site removed
- [x] Nginx configuration tested
- [x] Nginx reloaded
- [x] Backend reachable through Nginx
- [x] DNS verified
- [x] Health endpoint verified
- [x] WebSocket proxying supported
- [x] Nginx logs identified

---

# 38. Current Status

Phase 07 is complete.

The production API now has the following architecture:

```text
Internet
    |
    v
DNS
    |
    v
Nginx
    |
    +----------------+
    |                |
    v                v
HTTP API        WebSocket
    |                |
    +-------+--------+
            |
            v
     Docker Container
            |
            v
      Node.js Backend
            |
            v
       MongoDB Atlas
```

The application port:

```text
3000
```

is now behind Nginx.

The next step is HTTPS.

---

# 39. Remaining Improvements

The following are intentionally not considered complete yet:

```text
[ ] HTTPS / SSL
[ ] HTTP to HTTPS redirect
[ ] Certificate renewal verification
[ ] Production deployment documentation
[ ] Rollback testing
[ ] Deployment health check
[ ] Automatic rollback
[ ] DEV environment
[ ] TEST environment
[ ] Security hardening
[ ] Remove unnecessary public port 3000
[ ] Production secret management
[ ] Monitoring
[ ] Error/log management
[ ] MongoDB backup/recovery
```

These remain part of the overall project roadmap.

---

# 40. Next Phase

The next phase is:

```text
application/phase-08-domain-https.md
```

Phase 08 will document:

```text
Domain
   |
   v
DNS
   |
   v
EC2
   |
   v
Nginx
   |
   v
Let's Encrypt / SSL
   |
   v
HTTPS :443
   |
   v
Node.js :3000
```

It will cover:

- Domain setup
- DNS A record
- DNS verification
- HTTP verification
- Certbot
- Let's Encrypt certificate
- Nginx HTTPS configuration
- HTTP to HTTPS redirect
- Certificate renewal
- HTTPS health check
- WebSocket over HTTPS
- Troubleshooting
- Security considerations
- Lessons learned
- Verification checklist