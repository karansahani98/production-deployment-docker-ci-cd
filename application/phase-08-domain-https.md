# Phase 08 — Domain & HTTPS

## 1. Goal

The goal of this phase is to connect the production API to a real domain and secure the communication using HTTPS.

Before this phase:

```text
Internet
    |
    v
EC2
    |
    v
Nginx
    |
    v
Docker :3000
    |
    v
Node.js Backend
```

After this phase:

```text
Client
   |
   | HTTPS
   v
api.example.com
   |
   v
Nginx :443
   |
   v
Docker :3000
   |
   v
Node.js Backend
```

The final public API is accessed through:

```text
https://api.example.com
```

The actual production domain is intentionally excluded from this public repository.

---

# 2. Why Use a Domain?

An application can technically be accessed through an IP address:

```text
http://YOUR_EC2_IP:3000
```

However, this is not a good public API interface.

A domain provides:

```text
https://api.example.com
```

instead of:

```text
http://YOUR_EC2_IP:3000
```

Benefits include:

- Human-readable address
- Stable public endpoint
- HTTPS support
- Easier client configuration
- Easier migration to another server
- Better production architecture
- WebSocket support through a standard HTTPS endpoint

---

# 3. API Subdomain

The API uses a dedicated subdomain.

Example:

```text
api.example.com
```

The architecture is:

```text
example.com
     |
     +--> Website
     |
     +--> api.example.com
              |
              v
            API
```

The actual production domain is intentionally replaced with:

```text
api.example.com
```

in this documentation.

---

# 4. DNS Concept

DNS maps a hostname to an IP address.

Conceptually:

```text
api.example.com
       |
       | DNS
       v
YOUR_EC2_PUBLIC_IP
```

The API request then reaches:

```text
YOUR_EC2_PUBLIC_IP
       |
       v
EC2
```

The actual production IP address must never be stored in the public repository.

Use:

```text
YOUR_EC2_PUBLIC_IP
```

for examples.

---

# 5. DNS A Record

The API subdomain uses an A record.

Example:

```text
Type: A

Host:
api

Value:
YOUR_EC2_PUBLIC_IP
```

Conceptually:

```text
api.example.com
        |
        v
YOUR_EC2_PUBLIC_IP
```

This sends API traffic to the EC2 server.

---

# 6. Why Use an Elastic / Static Public IP?

A normal dynamically assigned public IP can change when infrastructure is recreated or certain instance lifecycle events occur.

A stable public IP is therefore useful for DNS.

The architecture becomes:

```text
DNS
 |
 v
Stable Public IP
 |
 v
EC2
```

If the EC2 instance changes internally, the public endpoint can remain stable when the static IP is retained and associated appropriately.

---

# 7. DNS Verification

After creating the DNS record, verify it from a client machine.

Windows:

```powershell
nslookup api.example.com
```

Expected concept:

```text
Name:
api.example.com

Address:
YOUR_EC2_PUBLIC_IP
```

The returned address should match the intended EC2 public IP.

---

# 8. DNS Propagation

DNS changes are not always visible everywhere immediately.

After changing DNS:

```text
DNS Provider
     |
     v
DNS Servers
     |
     v
Internet
```

Some clients may temporarily receive an older DNS result because of caching.

Therefore, when troubleshooting DNS:

```text
1. Check DNS record
2. Run nslookup
3. Verify returned IP
4. Verify EC2
5. Verify Nginx
```

Do not immediately assume the application is broken.

---

# 9. HTTP Before HTTPS

Before requesting an SSL certificate, the domain should first work over HTTP.

The expected architecture is:

```text
http://api.example.com
          |
          v
        Nginx
          |
          v
127.0.0.1:3000
          |
          v
       Backend
```

Test:

```bash
curl http://api.example.com/health
```

Expected:

```json
{
  "status": "ok"
}
```

This verifies that:

```text
DNS
+
EC2
+
Nginx
+
Docker
+
Node.js
```

are working together.

---

# 10. Why Verify HTTP First?

HTTPS adds another layer.

If HTTPS fails while HTTP has never been tested, there are too many possible failure points.

A better approach is:

```text
Step 1
Domain
   |
   v
HTTP
   |
   v
Backend
```

Then:

```text
Step 2
Domain
   |
   v
HTTPS
   |
   v
Backend
```

This isolates problems.

---

# 11. HTTP Nginx Configuration

The initial Nginx configuration listens on port:

```text
80
```

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

The actual production domain is intentionally not included.

---

# 12. HTTP Verification

After configuring Nginx:

```bash
sudo nginx -t
```

If successful:

```bash
sudo systemctl reload nginx
```

Then:

```bash
curl http://api.example.com/health
```

Expected:

```json
{
  "status": "ok"
}
```

At this point:

```text
HTTP
 |
 v
Nginx
 |
 v
Backend
```

is working.

---

# 13. Why HTTPS?

HTTP sends application traffic without transport encryption.

HTTPS adds TLS encryption between the client and the public endpoint.

The architecture becomes:

```text
Client
   |
   | Encrypted HTTPS
   v
Nginx
   |
   | Internal HTTP
   v
Node.js
```

The public connection is protected by TLS.

---

# 14. TLS Termination

In this architecture, Nginx handles TLS.

That means:

```text
Client
   |
   | HTTPS
   v
Nginx
   |
   | HTTP
   v
Node.js
```

Nginx decrypts the incoming HTTPS connection and forwards the request internally.

The Node.js application does not need to manage the public TLS certificate directly.

---

# 15. Let's Encrypt

A free TLS certificate was obtained using Let's Encrypt.

The certificate is managed using Certbot.

The conceptual flow is:

```text
Domain
   |
   v
Let's Encrypt
   |
   v
TLS Certificate
   |
   v
Nginx
   |
   v
HTTPS
```

---

# 16. Certbot Installation

Certbot and its Nginx integration were installed on the Ubuntu server.

Typical commands:

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

Verify:

```bash
certbot --version
```

---

# 17. Requesting the Certificate

Certbot can automatically configure Nginx.

Example:

```bash
sudo certbot --nginx -d api.example.com
```

The actual production domain is intentionally replaced with:

```text
api.example.com
```

Certbot validates domain ownership and configures the certificate.

---

# 18. Certificate Validation

The certificate request depends on the domain correctly pointing to the server.

The expected relationship is:

```text
api.example.com
       |
       v
YOUR_EC2_PUBLIC_IP
       |
       v
EC2
       |
       v
Nginx
       |
       v
Certbot
       |
       v
Let's Encrypt
```

If DNS is incorrect, certificate issuance can fail.

Therefore DNS should be verified before running Certbot.

---

# 19. HTTPS Nginx Configuration

After successful certificate setup, Nginx handles HTTPS.

Conceptually:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name api.example.com;

    ssl_certificate YOUR_CERTIFICATE_PATH;
    ssl_certificate_key YOUR_PRIVATE_KEY_PATH;

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

The actual certificate paths are managed by Certbot and are intentionally generalized here.

---

# 20. HTTP to HTTPS Redirect

After HTTPS is working, HTTP should redirect to HTTPS.

The desired flow is:

```text
http://api.example.com
          |
          v
       Nginx
          |
          | 301 / 308 redirect
          v
https://api.example.com
          |
          v
       Backend
```

This ensures clients use the encrypted endpoint.

The exact redirect configuration may be managed automatically by Certbot.

---

# 21. Why Redirect HTTP?

Without a redirect, both may exist:

```text
http://api.example.com
https://api.example.com
```

The goal is to establish one canonical public API endpoint:

```text
https://api.example.com
```

HTTP becomes only the entry point used to redirect clients to HTTPS.

---

# 22. HTTPS Health Check

After SSL configuration:

```bash
curl https://api.example.com/health
```

Expected:

```json
{
  "status": "ok"
}
```

This confirms:

```text
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
```

is working.

---

# 23. Windows HTTPS Verification

From PowerShell:

```powershell
curl.exe https://api.example.com/health
```

Using `curl.exe` explicitly avoids PowerShell's alias behavior.

Expected:

```json
{"status":"ok","uptime":...,"timestamp":"..."}
```

The actual production response may contain additional fields.

The important value is:

```text
status = ok
```

---

# 24. WebSocket Over HTTPS

The backend also uses WebSockets.

Once the API is available over HTTPS, the client should use secure WebSockets:

```text
wss://api.example.com/stream
```

instead of:

```text
ws://api.example.com/stream
```

The architecture becomes:

```text
Electron
    |
    | WSS
    v
Nginx :443
    |
    | WebSocket Upgrade
    v
Node.js
```

This is important because browser/Electron clients using a secure HTTPS environment should not depend on an insecure public WebSocket endpoint.

---

# 25. WebSocket Proxy Configuration

The Nginx configuration must retain:

```nginx
proxy_http_version 1.1;

proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

Without these headers, normal HTTPS requests may work while WebSocket connections fail.

The distinction is:

```text
HTTPS API
     |
     v
Works

WSS
     |
     v
Requires WebSocket Upgrade
```

---

# 26. Forwarded Protocol Header

The configuration includes:

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

When the client connects using:

```text
HTTPS
```

Nginx can communicate the original protocol to the backend.

Conceptually:

```text
Client
   |
   | HTTPS
   v
Nginx
   |
   | X-Forwarded-Proto: https
   v
Node.js
```

This is useful when application logic needs to know whether the original request was HTTP or HTTPS.

---

# 27. Certificate Renewal

Let's Encrypt certificates are short-lived and require renewal.

Certbot normally installs a renewal mechanism.

Check the timer:

```bash
sudo systemctl status certbot.timer
```

You can also inspect:

```bash
sudo certbot renew --dry-run
```

The dry run tests the renewal process without replacing the active certificate.

---

# 28. Why Test Renewal?

Having a valid certificate today does not guarantee that it will renew successfully later.

A production system should verify:

```text
Certificate
     |
     v
Renewal mechanism
     |
     v
Successful renewal
```

The goal is to avoid an unexpected certificate expiration.

---

# 29. Certificate Expiration

A certificate has an expiration date.

Check certificates with:

```bash
sudo certbot certificates
```

This shows:

```text
Certificate Name
Domains
Expiry Date
Certificate Path
Private Key Path
```

Do not publish real certificate paths or infrastructure information unnecessarily.

---

# 30. Security Group for HTTPS

The EC2 Security Group needs to permit:

```text
80
443
```

for the public web/API traffic required by the architecture.

Conceptually:

```text
Internet
   |
   +--> 80 HTTP
   |
   +--> 443 HTTPS
   |
   v
EC2
```

Port `3000` should not need to be publicly accessible once Nginx is the public entry point.

---

# 31. Removing Public Port 3000

The Docker Compose configuration initially exposes:

```yaml
ports:
  - "3000:3000"
```

This allows the host's port 3000 to access the application.

Once Nginx is correctly configured, the preferred architecture is to avoid public access to port `3000`.

The desired flow is:

```text
Internet
   |
   v
443
   |
   v
Nginx
   |
   v
3000
   |
   v
Backend
```

Port `3000` should be restricted to the server/internal network rather than exposed publicly.

This cleanup remains part of the security-hardening work.

---

# 32. Production API Endpoint

After this phase, the public API endpoint becomes:

```text
https://api.example.com
```

The client should use this endpoint instead of:

```text
http://YOUR_EC2_IP:3000
```

or:

```text
http://api.example.com
```

The HTTPS endpoint becomes the canonical production API URL.

---

# 33. Complete Request Flow

The final request flow is:

```text
                    Internet
                       |
                       | HTTPS :443
                       v
                api.example.com
                       |
                       v
                     Nginx
                       |
                       | reverse proxy
                       v
                127.0.0.1:3000
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

WebSocket flow:

```text
Electron
    |
    | WSS :443
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

# 34. Troubleshooting

## Problem 1 — Domain does not resolve

Check:

```powershell
nslookup api.example.com
```

Verify that the returned IP is:

```text
YOUR_EC2_PUBLIC_IP
```

If not, investigate DNS configuration.

---

## Problem 2 — HTTP does not work

Check:

```bash
sudo nginx -t
```

Then:

```bash
sudo systemctl status nginx
```

Then:

```bash
curl http://127.0.0.1:3000/health
```

If the direct backend request fails, investigate Docker/Node.js before investigating HTTPS.

---

## Problem 3 — HTTPS certificate request fails

Check:

```text
DNS
EC2 public IP
Port 80
Nginx
Domain name
```

The certificate authority must be able to validate the domain.

---

## Problem 4 — HTTPS works but API fails

Test:

```bash
curl https://api.example.com/health
```

Then check:

```bash
docker compose ps
```

and:

```bash
docker compose logs backend
```

Also verify:

```bash
sudo nginx -t
```

---

## Problem 5 — HTTP works but HTTPS does not

Check:

```bash
sudo systemctl status nginx
```

Then:

```bash
sudo nginx -t
```

Then inspect:

```bash
sudo tail -f /var/log/nginx/error.log
```

Also verify the Security Group allows:

```text
443
```

---

## Problem 6 — WebSocket fails after HTTPS

Verify the client uses:

```text
wss://
```

and Nginx contains:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

Then inspect backend logs:

```bash
docker compose logs -f backend
```

---

# 35. Layered Troubleshooting

The recommended debugging sequence is:

```text
1. EC2
     |
     v
2. Docker
     |
     v
3. Node.js
     |
     v
4. Nginx
     |
     v
5. DNS
     |
     v
6. HTTPS
     |
     v
7. WebSocket
```

Example:

### Backend

```bash
curl http://127.0.0.1:3000/health
```

### HTTP

```bash
curl http://api.example.com/health
```

### HTTPS

```bash
curl https://api.example.com/health
```

### WebSocket

```text
wss://api.example.com/stream
```

This prevents multiple layers from being debugged at the same time.

---

# 36. Security Considerations

## HTTPS

All public API communication should use HTTPS.

## Private Keys

TLS private keys must never be committed to Git.

## SSH Keys

EC2 SSH private keys must never be committed to Git.

## Secrets

Application secrets must remain outside source control.

## Public Ports

Only required public ports should be exposed.

## HTTP

HTTP should redirect to HTTPS after HTTPS is working.

---

# 37. Important Decisions

## Decision 1 — Use a dedicated API subdomain

Chosen:

```text
api.example.com
```

Reason:

```text
Website
    !=
API
```

This provides a clean separation between the website and backend API.

---

## Decision 2 — Use Nginx for TLS

Chosen:

```text
Client
   |
   | HTTPS
   v
Nginx
   |
   | HTTP
   v
Node.js
```

Reason:

- Centralized TLS termination
- No need to modify Node.js for certificate handling
- Easy integration with reverse proxy
- Supports WebSockets

---

## Decision 3 — Use Let's Encrypt

Chosen:

```text
Let's Encrypt
+
Certbot
```

Reason:

- Automated certificate issuance
- Automated renewal support
- No need for a manually purchased certificate for this deployment

---

## Decision 4 — Redirect HTTP to HTTPS

The production public endpoint should be:

```text
HTTPS
```

rather than leaving HTTP as a normal application endpoint.

---

## Decision 5 — Keep the real domain out of public documentation

The public repository uses:

```text
api.example.com
```

instead of the actual production domain.

This keeps the portfolio reusable and avoids unnecessary exposure of infrastructure details.

---

# 38. Lessons Learned

### Lesson 1

Verify DNS before debugging HTTPS.

### Lesson 2

Verify HTTP before introducing TLS.

### Lesson 3

Nginx can terminate TLS while Node.js continues running on port `3000`.

### Lesson 4

WebSocket traffic requires explicit upgrade configuration.

### Lesson 5

HTTPS API and WSS WebSocket endpoints should be treated as separate protocol requirements.

### Lesson 6

Certificate renewal should be tested before relying on automatic renewal.

### Lesson 7

The public application port should not remain unnecessarily exposed.

### Lesson 8

A layered troubleshooting approach is much faster than changing multiple infrastructure components simultaneously.

---

# 39. Verification Checklist

Before considering this phase complete:

- [x] Domain configured
- [x] API subdomain configured
- [x] DNS A record created
- [x] DNS resolution verified
- [x] EC2 public IP verified
- [x] HTTP endpoint verified
- [x] Nginx HTTP configuration verified
- [x] Certbot installed
- [x] Let's Encrypt certificate obtained
- [x] Nginx HTTPS configuration enabled
- [x] HTTPS endpoint verified
- [x] HTTP to HTTPS redirect configured
- [x] Certificate renewal mechanism configured
- [x] Certificate renewal dry run tested/planned
- [x] WebSocket proxy configuration retained
- [x] Secure WebSocket architecture documented
- [x] Production health endpoint verified over HTTPS

---

# 40. Current Status

Phase 08 is complete.

The production API now follows:

```text
Client
   |
   | HTTPS
   v
api.example.com
   |
   v
Nginx :443
   |
   v
Docker :3000
   |
   v
Node.js Backend
   |
   v
MongoDB Atlas
```

WebSocket:

```text
Electron
   |
   | WSS
   v
api.example.com
   |
   v
Nginx
   |
   v
Node.js WebSocket
```

The production API is now accessible through a domain secured with HTTPS.

---

# 41. Remaining Improvements

The following are intentionally not considered complete yet:

```text
[ ] Production deployment documentation
[ ] Rollback testing
[ ] Deployment health check inside CI
[ ] Automatic rollback
[ ] DEV environment
[ ] TEST environment
[ ] Security hardening
[ ] Remove unnecessary public port 3000
[ ] SSH hardening
[ ] File-type SSH CI variable
[ ] Production secret management
[ ] Monitoring
[ ] Error/log management
[ ] MongoDB backup/recovery
```

These remain part of the overall project roadmap.

---

# 42. Next Phase

The next phase is:

```text
application/phase-09-production-deployment.md
```

Phase 09 will document the complete production deployment process:

```text
Developer
   |
   | git push
   v
GitLab
   |
   v
CI/CD
   |
   v
Docker Image
   |
   v
Container Registry
   |
   v
Manual Production Deploy
   |
   v
EC2
   |
   v
Docker Compose
   |
   v
Nginx
   |
   v
HTTPS
   |
   v
Production API
```

It will connect all the previous phases into one complete deployment workflow and document the actual deployment verification process.