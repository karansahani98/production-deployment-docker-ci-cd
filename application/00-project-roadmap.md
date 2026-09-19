# Production Deployment — Project Roadmap

This document is the master progress tracker for the entire deployment journey.

The purpose is to record:

- What we planned
- What we actually implemented
- Commands used
- Important technical decisions
- Problems encountered
- Solutions
- Verification
- Remaining work
- Exact resume point for the next session

---

# 1. Project Goal

Build and document a production deployment workflow for a Node.js backend using:

- Node.js
- Docker
- Git
- GitLab
- GitLab Container Registry
- GitLab CI/CD
- AWS EC2
- MongoDB Atlas
- Nginx
- Custom domain
- HTTPS
- WebSocket support
- Production deployment
- Rollback
- Security hardening
- Monitoring

The public repository documents the engineering process.

The actual application repository remains private.

---

# 2. Repository Separation

There are two separate repositories.

## Private Application Repository

Contains:

- Real Node.js application
- Real source code
- Real environment configuration
- Real application secrets
- Production-specific configuration

This repository must remain private.

---

## Public Portfolio Repository

This repository contains:

- Deployment process
- Architecture
- Commands
- Sanitized configuration
- Troubleshooting
- Engineering decisions
- Lessons learned
- Deployment checklist
- Future roadmap

No real application secrets or private source code should be published here.

---

# 3. Security Rule

This public repository must NEVER contain:

```text
Real passwords
Real API keys
Real access tokens
Real SSH private keys
Real MongoDB credentials
Real JWT secrets
Real encryption keys
Real production .env files
Real private IP addresses
Real production IP addresses
Real production credentials


---

# Use placeholders:
YOUR_SECRET
YOUR_API_KEY
YOUR_PASSWORD
YOUR_MONGODB_URI
YOUR_EC2_IP
YOUR_DOMAIN
REGISTRY_IMAGE
YOUR_USERNAME


4. Target Architecture
The deployment architecture is:

                         Internet
                            |
                            v
                         Nginx
                       :80 / :443
                            |
                            v
                    Backend Docker
                            |
                            v
                       Node.js API
                            |
                            v
                       MongoDB Atlas

5. Deployment Philosophy

The project follows these principles.

Build Once

The application should be built into a Docker image once.

The same image can then be promoted between environments.

Source Code
    |
    v
Docker Build
    |
    v
Immutable Image
Immutable Image Versioning

Images should be identified using the Git commit SHA.

Example:

REGISTRY_IMAGE:<COMMIT_SHA>

instead of relying only on:

REGISTRY_IMAGE:latest

This allows us to identify exactly which source revision is running.

It also provides the foundation for rollback.

Manual Production Deployment

The current strategy is:

git push
    |
    v
CI automatically builds image
    |
    v
CI automatically pushes image
    |
    v
Production deployment waits for manual trigger

Production is therefore not automatically deployed on every push.

6. Phase Progress
Phase 01 — Local Development

Status:

✅ COMPLETED

Completed:

Node.js application verified
Dependencies installed
Application started locally
Health endpoint verified
Docker environment prepared
Local Docker image built
Docker container tested
MongoDB connectivity investigated
Docker-to-host MongoDB connectivity solved

Important lesson:

localhost inside a Docker container
does not refer to the Windows host.

For Docker Desktop local development:

host.docker.internal

can be used to reach the host.

7. Phase 02 — Git Workflow

Status:

✅ COMPLETED

Completed:

Git repository initialized
Main branch created
.gitignore configured
GitLab remote configured
Git authentication configured
Repository credentials troubleshot
Local repository pushed to GitLab
Main branch tracking configured

Important decision:

Repository authentication and container registry authentication were treated as separate concerns.

8. Phase 03 — Docker

Status:

✅ COMPLETED

Completed:

Dockerfile created
Node.js production image created
Dependencies installed during image build
Development dependencies excluded from production image
.dockerignore created
Port 3000 exposed
Production startup command configured
Docker image inspected
Container started
Container logs verified
API tested through Docker

Important decision:

A Debian-based slim Node.js image was used instead of Alpine because of native Node.js dependencies such as SQLite-related packages.

9. Phase 04 — Container Registry

Status:

✅ COMPLETED

Completed:

GitLab Container Registry enabled
Docker login tested
Docker image tagged
Docker image pushed
Registry image verified
CI registry authentication configured

Image strategy:

REGISTRY_IMAGE:<COMMIT_SHA>

The registry becomes the central location from which EC2 pulls production images.

10. Phase 05 — GitLab CI/CD

Status:

✅ COMPLETED

Completed:

.gitlab-ci.yml created
Docker-in-Docker configured
Docker image build automated
Commit SHA used as image tag
Image pushed automatically to registry
Production deployment job created
Production deployment configured as manual
SSH deployment from GitLab runner to EC2 configured
Deployment job successfully executed

Pipeline:

Build
  |
  v
Push Image
  |
  v
Manual Production Deployment

Important CI lesson:

The repository root must be understood correctly.

If the backend itself is the repository root, CI must not unnecessarily execute:

cd server
11. Phase 06 — AWS EC2

Status:

✅ COMPLETED

Completed:

AWS EC2 instance provisioned
Ubuntu configured
SSH access established
Docker installed
Docker Compose installed
Docker service verified
Docker usable without sudo
Application directory created
Production compose configuration created
Registry authentication configured
Production container started
12. Production Database

Status:

✅ COMPLETED

MongoDB Atlas was selected instead of running MongoDB directly on EC2.

Reason:

Application server
        |
        v
AWS EC2
        |
        v
MongoDB Atlas

The database therefore has a separate managed lifecycle from the application server.

Production connectivity was verified.

13. Phase 07 — Nginx

Status:

✅ COMPLETED

Completed:

Nginx installed
Reverse proxy configured
Backend routed through Nginx
HTTP request forwarding verified
WebSocket upgrade configuration added
Default Nginx site removed
Nginx configuration tested

Architecture:

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

Nginx is outside the Node.js Docker image.

14. Phase 08 — Domain + HTTPS

Status:

✅ COMPLETED

Completed:

API DNS record configured
DNS resolution verified
Nginx domain configuration created
Certbot installed
HTTPS certificate generated
Nginx HTTPS configuration completed
HTTPS health endpoint verified
Certificate renewal configured

Public documentation must use:

api.example.com

instead of the real production domain.

15. Phase 09 — Production Deployment

Status:

✅ COMPLETED

Completed:

Production image pulled from registry
Production environment configured
Docker Compose deployment verified
API health verified
Authentication flow verified
Production application communication verified
WebSocket path verified
Production deployment through GitLab CI verified

Deployment pattern:

IMAGE_TAG=<COMMIT_SHA> docker compose pull
IMAGE_TAG=<COMMIT_SHA> docker compose up -d
IMAGE_TAG=<COMMIT_SHA> docker compose ps

The IMAGE_TAG requirement is intentional.

16. Phase 10 — Rollback

Status:

⏳ NEXT

Planned:

Identify running image version
Keep previous known-good image version
Deploy previous image
Verify application health
Verify application logs
Test rollback through GitLab
Document rollback procedure
Test rollback after a simulated deployment failure

Future goal:

Failed Deployment
       |
       v
Health Check Fails
       |
       v
Automatic Rollback
       |
       v
Previous Known-Good Image
17. Phase 11 — Security Hardening

Status:

⏳ PLANNED

Remaining:

Rotate credentials exposed during development
Remove unnecessary public application port
Review AWS Security Group
Harden SSH access
Improve GitLab SSH private-key handling
Review registry credentials
Introduce production secret management
Review Docker runtime security
Review Nginx security configuration
18. Phase 12 — Monitoring & Operations

Status:

⏳ PLANNED

Remaining:

Application health monitoring
Container monitoring
CPU monitoring
Memory monitoring
Disk monitoring
Error monitoring
Log management
Deployment notifications
MongoDB backup/recovery
Resource monitoring
19. Testing

Status:

⏳ PLANNED

The initial CI pipeline intentionally focuses on:

Build
+
Registry Push
+
Manual Production Deployment

Automated tests need a proper environment because some integration/E2E tests require:

Running application
Database
Environment variables
Test data
WebSocket server

Future testing pipeline:

Unit Tests
    |
    v
Integration Tests
    |
    v
E2E Tests
    |
    v
Docker Build
    |
    v
Registry
    |
    v
Manual Production Deploy
20. Environment Strategy

Status:

⏳ PLANNED

Future environments:

DEV
 |
 v
TEST
 |
 v
PROD

Goals:

Environment-specific configuration
Environment-specific secrets
Same Docker image promoted between environments
Controlled production promotion
Clear deployment history

The exact infrastructure strategy will be decided later based on cost and operational requirements.

21. Current Production Architecture

The current backend production path is:

Client
  |
  | HTTPS / WebSocket
  v
Nginx
  |
  | reverse proxy
  v
Docker
  |
  v
Node.js Backend
  |
  v
MongoDB Atlas

CI/CD path:

Git Push
   |
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
Manual Deploy
   |
   v
AWS EC2
22. Important Problems Encountered
Docker → MongoDB

Problem:

Application works locally,
but database connection fails inside Docker.

Cause:

127.0.0.1 inside container != Windows host

Solution:

host.docker.internal

for Docker Desktop local development.

GitLab CI Repository Path

Problem:

CI attempted to enter a directory that did not exist in the CI working directory.

Cause:

The backend directory was already the repository root.

Solution:

Remove the unnecessary:

cd server
Compose IMAGE_TAG Requirement

Production Compose was intentionally changed to require an explicit image tag.

Without:

IMAGE_TAG

Compose fails.

This is expected behavior.

Deployment must explicitly identify the image version.

CI Deployment Verification

The deployment commands worked, but the final Compose status command also needed the image variable.

Correct pattern:

IMAGE_TAG=<COMMIT_SHA> docker compose ps

This was corrected in the CI pipeline.

23. Production Security Incident During Development

During development/troubleshooting, sensitive production configuration was accidentally exposed in a conversation.

The affected information must be treated as compromised.

Required future action:

Rotate affected credentials.

The public repository must not contain any of those values.

Do not copy sensitive command output into this repository.

24. Things Intentionally Not Included

This portfolio repository does not contain:

Real application source
Real .env
Production credentials
SSH private keys
Real API keys
Real database passwords
Real JWT secrets
Real infrastructure secrets

The repository demonstrates the engineering process without exposing the application.

25. Future Portfolio Work

After infrastructure work is complete:

 Finalize all phase documentation
 Add architecture diagrams
 Add sanitized screenshots where useful
 Add troubleshooting examples
 Add lessons learned
 Add deployment checklist
 Add rollback demonstration
 Review repository for secrets
 Publish public GitHub repository
 Add repository to LinkedIn
 Prepare LinkedIn project post
26. Resume Point
Current Phase
Phase 10 — Rollback
Last Completed Phase
Phase 09 — Production Deployment
Next Action

Implement and test rollback using a previously built commit-SHA image.

27. How to Continue This Project

When returning after a break:

1. Open this file.
2. Check the phase status.
3. Find the current resume point.
4. Open that phase document.
5. Verify the current infrastructure.
6. Continue implementation.
7. Record the commands used.
8. Record problems and solutions.
9. Update this roadmap.
10. Mark the phase complete only after verification.

This file is the single source of truth for the project's progress.


Save the file.

### Then verify

Run:

```powershell
Get-Content .\00-project-roadmap.md

Don't commit yet.