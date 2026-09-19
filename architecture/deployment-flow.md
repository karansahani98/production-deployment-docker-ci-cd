# Deployment Flow

## Overview

This document explains the complete deployment flow from local development to production.

The real application repository is private.

This public repository documents the generalized engineering process only.

No production credentials, private keys, real infrastructure values, or sensitive application source code should be placed here.

---

# 1. Complete Deployment Flow

The overall deployment process is:

Developer
    |
    | Code changes
    v
Local Development
    |
    | git commit
    v
Git Repository
    |
    | git push
    v
GitLab
    |
    | CI/CD Pipeline
    v
Docker Build
    |
    | Docker Image
    v
Container Registry
    |
    | Manual production deployment
    v
AWS EC2
    |
    | Docker Compose
    v
Application Container
    |
    v
Nginx
    |
    v
HTTPS / Domain
    |
    v
Users

---

# 2. Developer to Git

Development starts on the developer's local machine.

The developer:

1. Changes application code.
2. Runs the application locally.
3. Tests the changes.
4. Creates a Git commit.
5. Pushes the commit to GitLab.

Example:

    git status
    git add .
    git commit -m "Add feature"
    git push origin main

The Git repository becomes the source of truth for application code.

---

# 3. Git Push Triggers CI/CD

After the code is pushed:

Developer
    |
    v
GitLab Repository
    |
    v
Pipeline Trigger

GitLab CI/CD reads:

    .gitlab-ci.yml

The pipeline defines what should happen automatically.

The current deployment model separates:

- Build
- Deployment

Production deployment is intentionally manual.

---

# 4. Docker Build

The CI pipeline builds the application into a Docker image.

Conceptually:

Source Code
     |
     v
Dockerfile
     |
     v
Docker Build
     |
     v
Docker Image

The Docker image contains the application and its runtime dependencies.

Environment-specific secrets are not baked into the image.

---

# 5. Commit SHA Image Tag

Each build is associated with the Git commit SHA.

Conceptually:

Git Commit
    |
    | COMMIT_SHA
    v
Docker Image
    |
    | registry.example.com/application:COMMIT_SHA
    v
Container Registry

This creates a traceable relationship:

Commit
   ↓
Image
   ↓
Deployment

This is important for:

- Deployment tracking
- Debugging
- Rollback
- Reproducibility

---

# 6. Container Registry

After the image is built, CI pushes it to the Container Registry.

Conceptually:

GitLab CI
    |
    | docker push
    v
Container Registry

The registry stores versioned application images.

Example:

    registry.example.com/application:COMMIT_SHA

The production server does not need to build the application.

It pulls the already-built image.

---

# 7. Why Build on CI Instead of EC2

The production server should not normally build application images.

Preferred:

Developer
    |
    v
GitLab CI
    |
    +---- Build
    |
    +---- Package
    |
    v
Container Registry
    |
    v
EC2
    |
    +---- Pull image
    |
    +---- Run image

This separates:

Build responsibility

from:

Runtime responsibility

CI builds the artifact.

EC2 runs the artifact.

---

# 8. Manual Production Approval

The production deployment is intentionally manual.

The pipeline can reach:

Build
   |
   v
Image available
   |
   v
Manual Deploy

A production deployment is not automatically triggered by every push.

This gives the team an opportunity to:

- Review the commit
- Review the image
- Decide when to deploy
- Avoid accidental production releases

---

# 9. GitLab to EC2

When production deployment is manually triggered, GitLab CI connects to EC2 through SSH.

Conceptually:

GitLab CI
    |
    | SSH
    v
AWS EC2

The CI job uses protected CI/CD variables for deployment configuration.

Examples:

    EC2_HOST
    EC2_USER
    SSH_PRIVATE_KEY

Real values must never be committed to this repository.

---

# 10. EC2 Pulls the Image

The deployment process tells Docker Compose which image version to use.

Conceptually:

EC2
 |
 | docker compose pull
 v
Container Registry
 |
 | image
 v
EC2 Docker

The image is identified by the commit SHA.

Example:

    IMAGE_TAG=COMMIT_SHA

This avoids depending on a mutable latest tag for production deployment.

---

# 11. Docker Compose Starts the Application

After pulling the image:

    docker compose up -d

Docker Compose starts the backend container.

Conceptually:

EC2
 |
 +---- Docker
       |
       +---- Backend Container
               |
               +---- Node.js
               |
               +---- Port 3000

The production .env is supplied separately.

It is not stored in the public repository.

---

# 12. Runtime Configuration

The Docker image is environment-independent.

Runtime configuration is provided separately.

Conceptually:

Docker Image
     |
     +----------------------+
     |                      |
     v                      v
Development             Production
Environment             Environment
     |                      |
     v                      v
DEV Config              PROD Config

Examples of runtime configuration:

    MONGODB_URI
    JWT_SECRET
    API_KEYS
    PORT

Actual values must remain private.

---

# 13. Application Startup

The container starts the Node.js application.

Conceptually:

Docker Container
       |
       v
npm start
       |
       v
Node.js
       |
       +---- HTTP API
       |
       +---- WebSocket
       |
       +---- MongoDB
       |
       +---- External Services

The application listens on its internal application port.

Example:

    3000

---

# 14. Nginx Reverse Proxy

Nginx sits in front of the Node.js application.

The public request does not need to connect directly to the Node.js port.

Instead:

Internet
    |
    v
Nginx
    |
    v
Node.js :3000

Nginx provides the public HTTP/HTTPS entry point.

---

# 15. Domain Routing

The production API uses a domain.

Generalized example:

    api.YOUR_DOMAIN

DNS points the API hostname to the production server.

Conceptually:

api.YOUR_DOMAIN
       |
       v
   EC2 Public IP
       |
       v
      Nginx
       |
       v
   Node.js

The real domain should not be included in this public documentation.

---

# 16. HTTPS

Production requests use HTTPS.

Client
   |
   | HTTPS
   v
Nginx
   |
   | HTTP internally
   v
Node.js

TLS is terminated at Nginx.

This protects traffic between the client and public reverse proxy.

---

# 17. WebSocket Flow

The backend also supports WebSocket communication.

Production WebSocket traffic should use secure WebSockets.

Electron / Client
       |
       | WSS
       v
    Nginx
       |
       | WebSocket Upgrade
       v
    Node.js
       |
       v
Streaming Service

Nginx must forward the WebSocket upgrade headers.

---

# 18. Database Flow

MongoDB is hosted separately from the application server.

Conceptually:

Node.js
   |
   | MongoDB connection
   v
MongoDB Atlas

The database is not required to run inside the same Docker container as the Node.js application.

This keeps application runtime and database infrastructure separate.

---

# 19. External Service Flow

The backend may communicate with external services.

Examples:

Node.js
   |
   +---- MongoDB
   |
   +---- AI Provider
   |
   +---- Speech Provider
   |
   +---- Other External APIs

External API credentials remain server-side.

They should never be exposed to the browser or stored in public source code.

---

# 20. Complete Request Flow

A normal API request looks like:

User / Client
      |
      | HTTPS
      v
api.YOUR_DOMAIN
      |
      v
    Nginx
      |
      | Reverse Proxy
      v
Node.js Container
      |
      +---- Authentication
      |
      +---- Business Logic
      |
      +---- MongoDB
      |
      +---- External APIs
      |
      v
   Response
      |
      v
    Client

---

# 21. Complete Deployment Flow

The complete release process is:

1. Developer writes code.
2. Local testing.
3. Git commit.
4. Git push.
5. GitLab Pipeline.
6. Docker Build.
7. Tag with Commit SHA.
8. Push Image to Registry.
9. Manual Production Approval.
10. SSH to EC2.
11. Docker Compose Pull.
12. Docker Compose Up.
13. Application Starts.
14. Nginx Routes Traffic.
15. HTTPS Request.
16. Production Application.

---

# 22. Deployment Artifact

The important production artifact is:

    Docker Image

not:

    Developer's local source directory

The deployment model is:

Source Code
     |
     v
CI Build
     |
     v
Immutable Image
     |
     v
Registry
     |
     v
Production

This makes deployments reproducible.

---

# 23. Rollback Flow

If a deployment causes a problem:

Production Version B
       |
       v
Problem Detected
       |
       v
Identify Previous Version A
       |
       v
Pull Version A
       |
       v
Start Version A
       |
       v
Health Check
       |
       v
Production Restored

Because images are tagged with commit SHAs, the previous version can be identified precisely.

---

# 24. Why Commit SHA Matters

Without versioned images:

    latest
    latest
    latest

It becomes difficult to know exactly what was deployed.

With commit SHA:

    application:abc123
    application:def456
    application:789xyz

Each image represents a specific source revision.

This makes rollback and debugging easier.

---

# 25. Monitoring Flow

Production monitoring should observe the system from outside and inside.

External Monitor
       |
       | HTTPS
       v
 /health endpoint
       |
       v
    Nginx
       |
       v
   Node.js

The EC2 server can also be inspected for:

- CPU
- Memory
- Disk
- Docker
- Nginx
- Application Logs

---

# 26. Failure Detection

A production failure can occur at different layers.

DNS
 |
 +---- Failure
 |
HTTPS
 |
 +---- Failure
 |
Nginx
 |
 +---- Failure
 |
Docker
 |
 +---- Failure
 |
Node.js
 |
 +---- Failure
 |
MongoDB
 |
 +---- Failure
 |
External API

Troubleshooting should identify the failing layer before changing configuration.

---

# 27. Troubleshooting Flow

When production is unavailable:

Check public health endpoint
          |
          v
Check DNS
          |
          v
Check HTTPS
          |
          v
Check Nginx
          |
          v
Check Docker
          |
          v
Check Node.js logs
          |
          v
Check local health endpoint
          |
          v
Check MongoDB
          |
          v
Check external services

This creates a predictable troubleshooting process.

---

# 28. Security Boundaries

The deployment has multiple security boundaries:

Developer
    |
    v
GitLab Repository
    |
    v
GitLab CI
    |
    v
Container Registry
    |
    v
EC2
    |
    v
Nginx
    |
    v
Application
    |
    v
MongoDB / External APIs

Each boundary should use the minimum required access.

---

# 29. Separation of Responsibilities

The deployment architecture separates responsibilities.

### Git

Source control.

### GitLab CI/CD

Build and deployment automation.

### Container Registry

Artifact storage.

### EC2

Runtime infrastructure.

### Docker

Application isolation and runtime.

### Docker Compose

Container configuration and lifecycle.

### Nginx

Public reverse proxy and TLS termination.

### MongoDB Atlas

Database infrastructure.

### External Monitoring

Availability detection.

---

# 30. Build Once, Run Everywhere

The central deployment principle is:

Build Once
     |
     v
Immutable Image
     |
     +---- DEV
     |
     +---- TEST
     |
     +---- PROD

The application image should not be rebuilt separately for each environment.

Environment-specific configuration is supplied at runtime.

This reduces differences between environments.

---

# 31. Current Architecture

                         Internet
                            |
             +--------------+--------------+
             |                             |
             v                             v
      Public Website                  Electron App
      Future/Separate                Desktop Client
             |                             |
             +--------------+--------------+
                            |
                            v
                     Backend / API
                            |
                            v
                         Nginx
                            |
                            v
                    Docker / Node.js
                       /         \
                      /           \
                     v             v
             MongoDB Atlas    External APIs

The public website and Electron application have separate deployment lifecycles and should not be unnecessarily coupled to the backend deployment process.

---

# 32. Future Architecture Direction

As the system grows, the architecture may evolve.

Possible future architecture:

Load Balancer
     |
     +---- EC2 / Container 1
     |
     +---- EC2 / Container 2
     |
     +---- EC2 / Container 3

The current system intentionally starts simpler.

Infrastructure should grow according to actual requirements.

---

# 33. Important Design Decisions

### Decision 1 — Docker

The backend is packaged as a Docker image.

### Decision 2 — GitLab Registry

Images are stored in a container registry.

### Decision 3 — Commit SHA

Production images are versioned using the Git commit SHA.

### Decision 4 — Manual Production Deployment

Production deployment is manually triggered.

### Decision 5 — EC2

EC2 provides the initial production runtime.

### Decision 6 — Docker Compose

Docker Compose manages the production container.

### Decision 7 — Nginx

Nginx provides the public reverse-proxy layer.

### Decision 8 — HTTPS

Production API traffic uses HTTPS.

### Decision 9 — MongoDB Atlas

Database infrastructure is separated from the application host.

### Decision 10 — Public Documentation

The public repository documents the engineering process without exposing the real private application.

---

# 34. What This Architecture Solves

This deployment model provides:

- Repeatable builds
- Versioned artifacts
- Controlled production releases
- Easy image-based deployment
- Rollback capability
- HTTPS
- Reverse proxying
- Environment-specific runtime configuration
- Separation of source and runtime infrastructure
- Clear troubleshooting boundaries

---

# 35. What This Architecture Does Not Solve Yet

The following remain future implementation work:

- Full automated test pipeline
- Automated deployment health checks
- Automatic rollback
- DEV / TEST / PROD environment implementation
- Full monitoring and alerting
- Advanced centralized logging
- Backup/recovery validation
- Complete production security hardening
- Public website deployment
- Electron release/distribution

These are tracked separately in:

    application/99-pending-work.md

---

# 36. Final Mental Model

The easiest way to remember the deployment is:

CODE
  |
  v
GIT
  |
  v
CI
  |
  v
DOCKER IMAGE
  |
  v
REGISTRY
  |
  v
EC2
  |
  v
DOCKER
  |
  v
NGINX
  |
  v
HTTPS
  |
  v
USER

Application dependencies:

                Node.js
                /     \
               /       \
              v         v
       MongoDB Atlas   External APIs

The complete system:

Developer
    |
    v
GitLab
    |
    v
CI/CD
    |
    v
Container Registry
    |
    v
AWS EC2
    |
    v
Docker
    |
    v
Node.js
    |
    +--------> MongoDB Atlas
    |
    +--------> External APIs
    |
    v
Nginx
    |
    v
HTTPS
    |
    v
Users

---

# Summary

The deployment flow follows a simple and repeatable model:

1. Develop locally.
2. Commit code.
3. Push to GitLab.
4. Build a Docker image.
5. Tag the image with the commit SHA.
6. Push the image to the Container Registry.
7. Manually approve production deployment.
8. Connect to EC2 through SSH.
9. Pull the exact image version.
10. Start the container using Docker Compose.
11. Route traffic through Nginx.
12. Serve the API through HTTPS.
13. Connect the application to MongoDB Atlas and external services.
14. Monitor the production system.
15. Keep the previous image available for rollback.

The central principle is:

> Source code is versioned, Docker images are immutable artifacts, infrastructure runs those artifacts, and production configuration is supplied separately.