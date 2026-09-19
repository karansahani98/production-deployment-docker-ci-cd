# Phase 06 — AWS EC2

## 1. Goal

The goal of this phase is to document how the production backend infrastructure was created on AWS EC2.

The application is packaged as a Docker image.

GitLab CI builds and pushes that image to the GitLab Container Registry.

EC2 is responsible for running the application container.

The final responsibility is:

```text
GitLab
   |
   | Build
   v
Docker Image
   |
   | Push
   v
Container Registry
   |
   | Pull
   v
AWS EC2
   |
   v
Docker Container
   |
   v
Backend API
```

EC2 is therefore the production compute server.

---

# 2. Production Infrastructure Overview

The production architecture currently consists of:

```text
                         Internet
                            |
                            v
                    Domain / DNS
                            |
                            v
                         Nginx
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

The deployment path is separate:

```text
Developer
    |
    | git push
    v
GitLab Repository
    |
    v
GitLab CI/CD
    |
    | Docker build
    v
GitLab Container Registry
    |
    | docker pull
    v
AWS EC2
    |
    v
Docker Container
```

This separation is important.

The Git repository stores source code.

The Container Registry stores deployment artifacts.

EC2 runs the deployment artifact.

MongoDB Atlas provides the production database.

---

# 3. Why AWS EC2?

EC2 provides a virtual server where the application can run continuously.

For this project, the application already runs naturally as a Docker container.

Therefore the production model is straightforward:

```text
AWS EC2
   |
   +--> Docker
          |
          +--> Backend Container
```

The application does not need to know that it is running on EC2.

From the application's perspective:

```text
Node.js
   |
   v
Docker Container
```

The infrastructure handles:

```text
Internet
   |
   v
Nginx
   |
   v
Docker
   |
   v
Application
```

---

# 4. EC2 Instance

An Ubuntu-based EC2 instance was created for the backend.

The actual production instance information is intentionally not included in this public documentation.

Use placeholders such as:

```text
YOUR_EC2_INSTANCE_ID
YOUR_EC2_PUBLIC_IP
YOUR_EC2_PRIVATE_IP
YOUR_EC2_REGION
```

The production instance architecture was:

```text
AWS EC2
   |
   +--> Ubuntu Linux
   |
   +--> Docker
   |
   +--> Docker Compose
   |
   +--> Nginx
   |
   +--> Backend Container
```

---

# 5. Instance Sizing

The initial production instance was intentionally kept small.

The application was being deployed as a Dockerized backend and the goal was to keep infrastructure cost low during the initial deployment.

The instance size can be increased later if resource usage requires it.

The important principle is:

> Start with a reasonable infrastructure size and scale based on measured workload rather than assuming large infrastructure is required from day one.

Future scaling options include:

```text
Vertical Scaling
    |
    +--> Larger EC2 instance
```

or:

```text
Horizontal Scaling
    |
    +--> Multiple EC2 instances
    |
    +--> Load Balancer
```

The current architecture starts with a single EC2 instance.

---

# 6. Operating System

Ubuntu Linux was selected for the EC2 server.

The server is used as the infrastructure host.

The application itself runs inside Docker.

Therefore the separation is:

```text
Ubuntu
   |
   +--> Infrastructure
   |
   +--> Docker
          |
          +--> Application
```

This keeps application runtime dependencies inside the Docker image rather than installing Node.js and application dependencies directly on the server.

---

# 7. SSH Access

SSH is used for administrative access to EC2.

The private SSH key is stored locally and is never committed to the public repository.

Example:

```bash
ssh -i YOUR_SSH_KEY.pem YOUR_EC2_USER@YOUR_EC2_IP
```

For documentation:

```text
YOUR_SSH_KEY.pem
YOUR_EC2_USER
YOUR_EC2_IP
```

must always be placeholders.

Never commit:

```text
*.pem
*.key
id_rsa
id_ed25519
```

to a public repository.

---

# 8. SSH Key Permissions

The private key must be protected.

On Linux/macOS, this commonly means:

```bash
chmod 400 YOUR_SSH_KEY.pem
```

On Windows, SSH key permission handling can be different depending on the SSH client being used.

The important security principle is:

> The private SSH key must never be publicly accessible.

If the key is exposed, it should be treated as compromised and replaced.

---

# 9. Security Group

The EC2 Security Group controls inbound traffic to the server.

Conceptually:

```text
Internet
    |
    v
AWS Security Group
    |
    +--> SSH
    |
    +--> HTTP
    |
    +--> HTTPS
```

The exact production IP rules are intentionally omitted from this public repository.

Typical services required by the architecture are:

```text
SSH
HTTP
HTTPS
```

Application port `3000` does not need to be publicly exposed once Nginx is acting as the reverse proxy.

A future security cleanup should remove unnecessary public access to the backend application port.

---

# 10. Why Nginx Instead of Public Port 3000?

The Node.js application listens internally on:

```text
3000
```

Initially, Docker maps:

```text
3000:3000
```

However, exposing the application port directly to the internet is not necessary when Nginx is used.

The desired architecture is:

```text
Internet
   |
   | HTTPS :443
   v
Nginx
   |
   | HTTP
   v
127.0.0.1:3000
   |
   v
Docker Container
```

Therefore:

```text
Public
   |
   v
443
   |
   v
Nginx
   |
   v
3000
```

Port `3000` becomes an internal application port rather than the public API entry point.

Nginx and HTTPS are documented in the following phases.

---

# 11. Installing Docker

Docker was installed on the Ubuntu EC2 instance.

The goal is to run the production backend using the same container model used during local development.

After installation, verify:

```bash
docker --version
```

Expected:

```text
Docker version ...
```

Then verify the Docker service:

```bash
sudo systemctl status docker
```

The service should be active.

---

# 12. Docker Service

Docker should start automatically after the server boots.

Verify:

```bash
sudo systemctl is-enabled docker
```

If required:

```bash
sudo systemctl enable docker
```

The production architecture depends on Docker being available after a server restart.

---

# 13. Docker Permissions

Initially, Docker commands may require:

```bash
sudo docker ...
```

For normal operational use, the EC2 user was added to the Docker group.

Example:

```bash
sudo usermod -aG docker $USER
```

After changing group membership, reconnect to SSH.

Then verify:

```bash
docker ps
```

The goal is to allow the deployment user to execute Docker commands without requiring `sudo`.

---

# 14. Important Docker Permission Lesson

Adding a user to the Docker group changes the user's privileges significantly.

Membership in the Docker group effectively provides powerful control over the host.

Therefore:

```text
docker group
     |
     v
High privilege
```

This should be considered when designing production access.

The deployment user should not be given more access than necessary.

---

# 15. Docker Compose

Docker Compose is used to define the production backend container.

The production Compose configuration conceptually looks like:

```yaml
services:
  backend:
    image: registry.example.com/project/backend:${IMAGE_TAG:?IMAGE_TAG is required}

    container_name: backend

    restart: unless-stopped

    env_file:
      - .env

    ports:
      - "3000:3000"
```

The actual registry and project values are intentionally replaced with placeholders.

---

# 16. Why Docker Compose?

The application currently consists of a small number of infrastructure components.

Docker Compose provides a simple way to define:

```text
Container
Image
Environment
Ports
Restart Policy
```

The deployment command becomes:

```bash
docker compose up -d
```

Instead of manually running a long Docker command every time.

---

# 17. Production Application Directory

A dedicated directory was created on EC2 for the backend deployment.

Example:

```text
/home/ubuntu/support-assistant/backend/
```

For public documentation, use:

```text
/home/YOUR_EC2_USER/YOUR_APPLICATION/backend/
```

The directory contains deployment-related files such as:

```text
backend/
   |
   +--> docker-compose.yml
   |
   +--> .env
```

The actual application source code does not need to be copied to EC2 for the container deployment model.

This is important.

EC2 receives the Docker image rather than building the source code itself.

---

# 18. Production .env

The production application requires environment variables.

These include application configuration and external service credentials.

The production `.env` file exists only on the server.

It must not be committed to Git.

Example:

```text
.env
```

should be:

```text
server only
```

and not:

```text
Git repository
```

The public documentation must only show placeholders:

```env
PORT=3000
MONGODB_URI=YOUR_MONGODB_URI
JWT_SECRET=YOUR_JWT_SECRET
API_KEY=YOUR_API_KEY
```

Never document real values.

---

# 19. MongoDB Architecture

MongoDB is not hosted directly on the EC2 server.

The production database uses MongoDB Atlas.

Therefore:

```text
EC2
 |
 | MongoDB connection
 v
MongoDB Atlas
```

The backend container connects to MongoDB using:

```text
MONGODB_URI
```

This is supplied through the production environment.

---

# 20. MongoDB Network Access

MongoDB Atlas uses network access controls.

The EC2 public IP must be allowed by the Atlas network configuration for the backend to connect.

Conceptually:

```text
EC2
 |
 | Public IP
 v
MongoDB Atlas Network Access
 |
 v
MongoDB
```

The actual production IP is intentionally not documented publicly.

Use:

```text
YOUR_EC2_PUBLIC_IP
```

in examples.

---

# 21. MongoDB Credentials

The database username and password are not stored in:

```text
docker-compose.yml
```

and should not be stored in:

```text
.gitlab-ci.yml
```

They belong in protected secret storage.

For the current implementation, the production application receives them through:

```text
.env
```

on EC2.

A future improvement is to use dedicated production secret management.

---

# 22. Connecting EC2 to GitLab Container Registry

The EC2 server needs permission to pull private Docker images.

The basic flow is:

```text
GitLab Container Registry
          |
          | docker pull
          v
         EC2
```

EC2 authenticates with the GitLab Container Registry.

Conceptually:

```bash
docker login YOUR_REGISTRY
```

Then:

```bash
docker pull YOUR_REGISTRY_IMAGE:YOUR_IMAGE_TAG
```

The actual registry credentials must never be documented.

---

# 23. Registry Credentials

Registry credentials are separate from Git repository credentials.

Conceptually:

```text
Git Repository Access
        |
        +--> Source code

Container Registry Access
        |
        +--> Docker images
```

The credentials should have only the permissions required for their purpose.

For production, the EC2 server only needs the ability to pull images.

Therefore registry access should ideally be limited to pull/read permissions where supported.

---

# 24. Initial Production Deployment

The initial deployment was performed manually before the CI/CD deployment process was fully automated.

The basic flow was:

```text
Developer
    |
    v
Docker Image
    |
    v
GitLab Container Registry
    |
    v
EC2
    |
    v
docker compose pull
    |
    v
docker compose up -d
```

This was useful because it verified that:

```text
Docker image
+
EC2
+
MongoDB
+
Application
```

could work together before automating deployment.

---

# 25. Pulling the Application Image

Once the image is available in the registry:

```bash
docker pull YOUR_REGISTRY_IMAGE:YOUR_IMAGE_TAG
```

For the current architecture, the preferred image version is based on:

```text
CI_COMMIT_SHA
```

rather than:

```text
latest
```

This gives the server a deterministic image version.

---

# 26. Starting the Container

After the image is available:

```bash
IMAGE_TAG=YOUR_IMAGE_TAG docker compose up -d
```

The container can then be checked with:

```bash
docker compose ps
```

Expected concept:

```text
NAME                    STATUS
backend                 running
```

---

# 27. Container Logs

If the application does not start:

```bash
docker compose logs backend
```

For live logs:

```bash
docker compose logs -f backend
```

This is one of the most important production debugging commands.

Typical problems can include:

```text
Missing environment variable
Database connection failure
Port conflict
Invalid image
Application startup failure
```

---

# 28. Checking Running Containers

Use:

```bash
docker ps
```

This shows running containers.

For all containers:

```bash
docker ps -a
```

Useful information includes:

```text
Container ID
Image
Status
Ports
Container Name
```

---

# 29. Checking the Image

Use:

```bash
docker images
```

or:

```bash
docker image ls
```

This helps verify that the expected image exists locally on EC2.

With commit-based deployment, the expected tag should be visible.

---

# 30. Application Health Check

Before putting the API behind the domain, verify the application directly.

For example:

```bash
curl http://127.0.0.1:3000/health
```

Expected:

```json
{
  "status": "ok"
}
```

This verifies:

```text
Docker
   |
   v
Node.js
   |
   v
Application
```

without involving Nginx or DNS.

This is an important troubleshooting technique.

Test from the bottom layer upward:

```text
1. Container
2. Application
3. Nginx
4. Domain
5. HTTPS
```

---

# 31. MongoDB Connection Verification

The application should successfully connect to MongoDB during startup.

The application logs can be checked with:

```bash
docker compose logs backend
```

A successful startup should indicate that the application initialized correctly and connected to its required dependencies.

If MongoDB connection fails, investigate:

```text
MONGODB_URI
Atlas Network Access
Database credentials
DNS/network connectivity
TLS configuration
```

Do not expose the actual MongoDB URI while debugging publicly.

---

# 32. EC2 and GitLab CI/CD

After CI/CD was configured, EC2 became the deployment target.

The flow is:

```text
Developer
    |
    | git push
    v
GitLab
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
GitLab Deploy Job
    |
    | SSH
    v
EC2
    |
    v
docker compose pull
    |
    v
docker compose up -d
```

EC2 therefore does not need to build the Docker image.

---

# 33. Why EC2 Does Not Build the Application

A common deployment pattern is:

```text
EC2
 |
 +--> git pull
 |
 +--> npm install
 |
 +--> npm build
 |
 +--> npm start
```

That is not the chosen architecture.

Instead:

```text
GitLab CI
 |
 +--> Build Docker Image
 |
 +--> Push Image
 |
 v
Registry
 |
 v
EC2
 |
 +--> Pull Image
 |
 +--> Run Container
```

This keeps the production server focused on running the application.

---

# 34. Build Once, Run Anywhere

The Docker image contains the application runtime and dependencies.

Therefore:

```text
Developer Environment
       |
       v
Docker Image
       |
       +------> DEV
       |
       +------> TEST
       |
       +------> PROD
```

The environment-specific values are supplied through environment configuration.

The application artifact itself remains the same.

---

# 35. EC2 Restart Behaviour

The production Compose configuration uses:

```yaml
restart: unless-stopped
```

This means Docker can restart the application container after certain failures or Docker daemon restarts.

Conceptually:

```text
Server
   |
   | reboot
   v
Docker
   |
   v
Container
   |
   v
Backend
```

This does not replace proper monitoring or deployment health checks.

It only provides basic container restart behavior.

---

# 36. Production Deployment Boundary

The current architecture has a clear responsibility split:

```text
GitLab
   |
   +--> Source Code
   +--> CI/CD
   +--> Image Build
   +--> Image Registry
```

and:

```text
EC2
   |
   +--> Docker
   +--> Running Container
   +--> Nginx
```

and:

```text
MongoDB Atlas
   |
   +--> Production Database
```

This separation makes each component easier to reason about.

---

# 37. Troubleshooting Approach

When production is not working, do not immediately assume the application is broken.

Debug from the bottom upward.

## Step 1 — Check EC2

```bash
uptime
```

Verify the server is running.

## Step 2 — Check Docker

```bash
docker ps
```

## Step 3 — Check Container

```bash
docker compose ps
```

## Step 4 — Check Logs

```bash
docker compose logs backend
```

## Step 5 — Check Application Directly

```bash
curl http://127.0.0.1:3000/health
```

## Step 6 — Check Nginx

```bash
sudo nginx -t
```

## Step 7 — Check Domain

```bash
curl https://YOUR_DOMAIN/health
```

This layered approach makes troubleshooting much faster.

---

# 38. Common Problems Encountered

## Problem 1 — Docker required sudo

Initially:

```bash
docker ps
```

could require:

```bash
sudo docker ps
```

### Fix

Add the deployment user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Reconnect to SSH.

Then:

```bash
docker ps
```

---

## Problem 2 — Application container could not connect to MongoDB

The application requires:

```text
MONGODB_URI
```

If the variable is missing or incorrect, startup can fail.

### Fix

Verify the production environment configuration on EC2 without exposing its values.

Then verify MongoDB Atlas network access.

---

## Problem 3 — EC2 cannot pull private image

If:

```bash
docker pull YOUR_REGISTRY_IMAGE:YOUR_IMAGE_TAG
```

fails, verify:

```text
Registry authentication
Image name
Image tag
Network connectivity
Registry permissions
```

---

## Problem 4 — Wrong image version

Using:

```text
latest
```

can make it difficult to know which version is running.

### Fix

Use:

```text
CI_COMMIT_SHA
```

as the deployment image tag.

---

# 39. Infrastructure Security

The EC2 server should follow the principle of least exposure.

Publicly required services should be limited to what the architecture needs.

Conceptually:

```text
Internet
   |
   +--> 443 HTTPS
   |
   +--> 80 HTTP
   |
   +--> SSH
```

The backend application port:

```text
3000
```

should not need to be directly accessible from the public internet once Nginx is configured correctly.

---

# 40. SSH Security

SSH is an administrative access channel.

Future hardening should include:

```text
[ ] Restrict SSH source IP where practical
[ ] Disable unnecessary authentication methods
[ ] Use key-based authentication
[ ] Review SSH configuration
[ ] Review active users
[ ] Remove unnecessary access
```

These tasks belong to the security phase.

---

# 41. Secrets Security

Production secrets should never be stored in:

```text
Git
Dockerfile
docker-compose.yml
README.md
GitLab CI YAML
Public documentation
```

Instead, use appropriate secret storage.

Current implementation:

```text
EC2
 |
 +--> .env
```

Future improvement:

```text
Secret Management
        |
        v
Application
```

The actual secret-management solution will be evaluated during the security phase.

---

# 42. Important Infrastructure Decision

The application source code does not need to be stored on EC2.

Instead:

```text
GitLab
   |
   v
Docker Image
   |
   v
Registry
   |
   v
EC2
```

This is an important architectural decision.

EC2 is a runtime environment rather than the source-code build environment.

---

# 43. Current Production Architecture

At this stage, the production infrastructure can be represented as:

```text
                         Internet
                            |
                            v
                       DNS / Domain
                            |
                            v
                          Nginx
                            |
                            v
                    Docker Container
                            |
                            v
                      Node.js API
                            |
                            v
                      MongoDB Atlas
```

Deployment:

```text
Developer
    |
    v
GitLab Repository
    |
    v
GitLab CI/CD
    |
    v
Docker Build
    |
    v
GitLab Container Registry
    |
    v
AWS EC2
    |
    v
Docker Compose
    |
    v
Backend Container
```

---

# 44. Verification Checklist

Before considering this phase complete:

- [x] AWS EC2 instance created
- [x] Ubuntu server configured
- [x] SSH access verified
- [x] Elastic/public IP configured
- [x] Security Group configured
- [x] Docker installed
- [x] Docker service running
- [x] Docker permissions configured
- [x] Docker Compose installed
- [x] Production application directory created
- [x] Production environment configuration created
- [x] MongoDB Atlas connectivity configured
- [x] GitLab Container Registry connectivity configured
- [x] Docker image pulled to EC2
- [x] Backend container started
- [x] Container status verified
- [x] Application logs verified
- [x] Local health endpoint verified
- [x] CI/CD deployment connected to EC2

---

# 45. What Runs Where?

A useful way to understand the architecture is:

| Component | Location |
|---|---|
| Source Code | GitLab Repository |
| CI/CD | GitLab CI |
| Docker Image | GitLab Container Registry |
| Backend Runtime | AWS EC2 |
| Docker | AWS EC2 |
| Nginx | AWS EC2 |
| Database | MongoDB Atlas |
| Production Secrets | Server-side configuration |
| DNS | Domain/DNS Provider |

The actual infrastructure identifiers are intentionally excluded from the public repository.

---

# 46. Important Lessons

### Lesson 1

EC2 should run the application, not build it.

### Lesson 2

Docker provides consistency between development and production.

### Lesson 3

The Container Registry acts as the bridge between CI/CD and EC2.

### Lesson 4

A production server should expose as little as possible publicly.

### Lesson 5

Application debugging should be performed layer by layer.

### Lesson 6

MongoDB does not need to run on the same EC2 instance.

### Lesson 7

Production secrets must remain outside source control.

### Lesson 8

Commit-based image tags make deployments traceable.

### Lesson 9

Docker restart policies help with basic recovery but do not replace monitoring.

---

# 47. Current Status

Phase 06 is complete.

The production compute infrastructure is now represented as:

```text
AWS EC2
   |
   +--> Ubuntu
   |
   +--> Docker
   |
   +--> Docker Compose
   |
   +--> Backend Container
   |
   +--> Nginx
```

The deployment relationship is:

```text
GitLab CI
    |
    v
Container Registry
    |
    v
EC2
    |
    v
Docker
    |
    v
Backend
```

The database relationship is:

```text
Backend
    |
    v
MongoDB Atlas
```

---

# 48. Remaining Improvements

The following are intentionally not considered complete yet:

```text
[ ] Nginx reverse proxy documentation
[ ] Domain configuration documentation
[ ] HTTPS / SSL documentation
[ ] Production deployment documentation
[ ] Rollback testing
[ ] Deployment health check
[ ] Automatic rollback
[ ] DEV environment
[ ] TEST environment
[ ] Security hardening
[ ] SSH hardening
[ ] Remove unnecessary public port 3000
[ ] Production secret management
[ ] Monitoring
[ ] Error/log management
[ ] MongoDB backup/recovery
```

These remain part of the overall project roadmap.

---

# 49. Next Phase

The next phase is:

```text
application/phase-07-nginx.md
```

Phase 07 will document:

```text
Internet
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

It will cover:

- Why Nginx was introduced
- Reverse proxy concept
- Nginx installation
- Nginx server block
- Domain routing
- Proxy headers
- WebSocket proxying
- HTTP to HTTPS preparation
- Nginx configuration testing
- Enabling the site
- Removing the default site
- Troubleshooting
- Verification
- Security considerations
- Lessons learned
- Current status