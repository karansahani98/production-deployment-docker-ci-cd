# Production Architecture

## Overview

This document describes the production architecture used for the backend deployment.

The architecture is intentionally simple and cost-conscious while providing:

- Dockerized application runtime
- GitLab CI/CD
- Container Registry
- AWS EC2
- Nginx reverse proxy
- HTTPS
- MongoDB Atlas
- External API integrations
- WebSocket communication
- Rollback capability
- Environment-based configuration

The actual application repository and production infrastructure remain private.

This document uses generalized names and placeholders.

---

# 1. High-Level Architecture

The production architecture is:

                         Internet
                            |
             +--------------+--------------+
             |                             |
             v                             v
      Public Website                  Electron App
      Separate Lifecycle             Desktop Client
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

The backend is the central application service.

The public website and Electron application are separate clients and have separate deployment lifecycles.

---

# 2. Main Components

The production system contains the following major components:

1. Client applications
2. Domain / DNS
3. Nginx
4. Docker
5. Node.js backend
6. MongoDB Atlas
7. External service providers
8. GitLab repository
9. GitLab CI/CD
10. Container Registry
11. AWS EC2
12. Monitoring

---

# 3. Client Layer

There are multiple possible clients.

## Electron Application

The Electron application acts as a desktop client.

Conceptually:

    Electron App
         |
         | HTTPS / WSS
         v
    Backend API

The Electron application does not need direct access to MongoDB.

The backend remains responsible for:

- Authentication
- Authorization
- Business logic
- Database access
- External API communication

---

## Public Website

The public website has a separate responsibility.

It may provide:

- SEO content
- Product information
- Basic pages
- User registration

It should not be unnecessarily coupled to the backend deployment lifecycle.

The website can communicate with the backend through HTTPS APIs where required.

---

# 4. Internet Entry Point

Public traffic enters through the domain.

Generalized example:

    https://api.YOUR_DOMAIN

The DNS record points the API hostname to the production server.

Conceptually:

    Client
       |
       v
    DNS
       |
       v
    EC2 Public IP
       |
       v
    Nginx

The actual production domain and IP must not be stored in this public repository.

---

# 5. HTTPS Layer

Production API traffic uses HTTPS.

The request flow is:

    Client
       |
       | HTTPS
       v
    Nginx
       |
       | HTTP
       v
    Node.js

TLS is terminated at Nginx.

This protects data travelling between clients and the public server.

Examples of protected information include:

- Login credentials
- Authentication tokens
- API requests
- API responses
- User information
- WebSocket communication

---

# 6. Nginx Reverse Proxy

Nginx is the public reverse proxy.

Its responsibility is to receive public requests and forward them to the backend container.

Conceptually:

    Internet
       |
       v
    Nginx :443
       |
       v
    Node.js :3000

Nginx also handles:

- HTTPS
- Domain routing
- Reverse proxying
- Request headers
- WebSocket upgrade

---

# 7. Why Nginx Is Used

Without a reverse proxy:

    Internet
       |
       v
    Node.js :3000

With Nginx:

    Internet
       |
       v
    HTTPS
       |
       v
    Nginx
       |
       v
    Node.js :3000

This creates a controlled public boundary.

The Node.js application does not need to directly handle public TLS termination.

---

# 8. Docker Layer

The backend runs inside a Docker container.

Conceptually:

    EC2
      |
      v
    Docker
      |
      v
    Backend Container
      |
      v
    Node.js

Docker provides a consistent runtime environment.

The application image contains:

- Node.js runtime
- Production dependencies
- Application source
- Application startup configuration

Environment-specific secrets are provided separately.

---

# 9. Docker Image

The production application is packaged as a Docker image.

Generalized example:

    registry.example.com/application:COMMIT_SHA

The image is built by GitLab CI/CD.

Production EC2 does not need to build the image.

Instead:

    CI
      |
      v
    Build Image
      |
      v
    Registry
      |
      v
    EC2 Pulls Image

---

# 10. Immutable Image Strategy

Production deployments use commit SHA image tags.

Example:

    application:abc123

The next version may be:

    application:def456

These are separate artifacts.

The benefit is that the production server can run an exact version.

This provides:

- Traceability
- Reproducibility
- Easier debugging
- Easier rollback

---

# 11. Container Registry

The Container Registry stores application images.

Conceptually:

    GitLab CI
        |
        | docker push
        v
    Container Registry
        |
        | docker pull
        v
    EC2

The registry is the bridge between:

    Build Infrastructure

and:

    Runtime Infrastructure

---

# 12. AWS EC2

AWS EC2 provides the production compute environment.

The EC2 server runs:

- Ubuntu
- Docker
- Docker Compose
- Nginx
- Backend container

Conceptually:

    AWS EC2
       |
       +---- Nginx
       |
       +---- Docker
               |
               +---- Backend Container

---

# 13. Why EC2

The initial deployment uses EC2 because it provides:

- Direct server control
- Simple Docker deployment
- Straightforward networking
- Low initial infrastructure complexity
- Flexible configuration
- Predictable runtime environment

The architecture can evolve later if the workload requires additional infrastructure.

---

# 14. Docker Compose

Docker Compose manages the backend container.

Conceptually:

    docker-compose.yml
           |
           v
       Docker Compose
           |
           v
      Backend Container

Compose defines:

- Image
- Container name
- Restart policy
- Environment configuration
- Port mapping

The current production deployment uses Compose rather than a larger orchestration platform.

---

# 15. Application Port

The Node.js backend listens on:

    3000

The desired production traffic path is:

    Internet
       |
       v
    Nginx :443
       |
       v
    Node.js :3000

Port 3000 is an application/runtime port.

It should not be unnecessarily exposed directly to the public Internet.

---

# 16. Network Architecture

Conceptually:

    Internet
       |
       | 80 / 443
       v
    AWS EC2
       |
       +---- Nginx
       |
       +---- Docker
               |
               +---- Node.js :3000
                       |
                       +---- MongoDB Atlas
                       |
                       +---- External APIs

Public traffic enters through Nginx.

The application communicates outward to required dependencies.

---

# 17. Security Group

The AWS Security Group controls network access to EC2.

The intended public application ports are:

    80
    443

SSH:

    22

should be restricted as much as practical.

Application port:

    3000

should not be publicly accessible when Nginx is the public entry point.

The exact Security Group configuration should be reviewed periodically.

---

# 18. MongoDB Atlas

MongoDB is hosted separately using MongoDB Atlas.

The backend connects to MongoDB through the configured connection string.

Conceptually:

    Node.js
       |
       | MongoDB connection
       v
    MongoDB Atlas

This separates:

    Application Compute

from:

    Database Infrastructure

---

# 19. MongoDB Network Access

MongoDB Atlas should allow only the required application infrastructure to connect.

Conceptually:

    EC2
      |
      | Allowed connection
      v
    MongoDB Atlas

Avoid unnecessarily exposing the database to the entire Internet.

---

# 20. Database Credentials

MongoDB credentials are runtime secrets.

They are supplied through environment configuration.

Example:

    MONGODB_URI=YOUR_MONGODB_URI

The actual value must never be placed in:

- Git
- Dockerfile
- Public README
- Public documentation
- CI configuration
- Screenshots
- Logs

---

# 21. External APIs

The backend communicates with external services when required.

Conceptually:

                    +---- MongoDB Atlas
                    |
    Node.js Backend-+---- AI Provider
                    |
                    +---- Speech Provider
                    |
                    +---- Other APIs

The backend is responsible for securely communicating with these services.

API credentials remain server-side.

---

# 22. External API Failure

External services are dependencies.

They can fail independently of the application.

Possible failures include:

- Timeout
- Rate limiting
- Authentication failure
- Provider outage
- Invalid request
- Network failure

The application should handle these failures gracefully.

A provider failure should not automatically expose internal details to the client.

---

# 23. WebSocket Architecture

The backend supports WebSocket communication.

Production flow:

    Electron / Client
          |
          | WSS
          v
        Nginx
          |
          | Upgrade
          v
       Node.js
          |
          v
    Streaming Provider

Nginx must support WebSocket upgrade headers.

Secure WebSockets should be used in production.

---

# 24. WebSocket Authentication

WebSocket connections should be authenticated.

Conceptually:

    Client
       |
       | Authentication
       v
    Backend
       |
       +---- Validate identity
       |
       +---- Validate session
       |
       v
    WebSocket Connection

WebSocket messages should also be validated.

---

# 25. Application Layer

The Node.js application is responsible for:

- HTTP APIs
- Authentication
- JWT handling
- Business logic
- Database access
- WebSocket communication
- External API integration
- Error handling
- Logging

The application should not expose internal infrastructure details to clients.

---

# 26. Runtime Configuration

Production configuration is injected at runtime.

Example:

    Node.js Container
          |
          +---- PORT
          +---- MONGODB_URI
          +---- JWT_SECRET
          +---- API_KEYS
          +---- Other configuration

The values are environment-specific.

The Docker image remains independent of these values.

---

# 27. Secret Flow

Secrets should follow this pattern:

    Secret Store / Protected Configuration
                  |
                  v
          Runtime Environment
                  |
                  v
           Docker Container
                  |
                  v
              Node.js

Not:

    Secret
      |
      v
    Git Repository
      |
      v
    Docker Image
      |
      v
    Production

Secrets should never be baked into source code or images.

---

# 28. CI/CD Architecture

The CI/CD flow is:

    Developer
       |
       | git push
       v
    GitLab
       |
       v
    GitLab CI/CD
       |
       +---- Docker Build
       |
       +---- Docker Push
       |
       v
    Container Registry
       |
       | Manual production deployment
       v
    EC2
       |
       v
    Docker Compose

---

# 29. CI/CD Build Stage

The build stage:

1. Starts the Docker build environment.
2. Logs into the Container Registry.
3. Builds the Docker image.
4. Tags the image with the commit SHA.
5. Pushes the image to the registry.

Conceptually:

    Git Commit
        |
        v
    CI Build
        |
        v
    application:COMMIT_SHA
        |
        v
    Registry

---

# 30. CI/CD Deployment Stage

Production deployment is manually triggered.

Conceptually:

    Build Passed
        |
        v
    Image Available
        |
        v
    Manual Deploy
        |
        v
    SSH -> EC2
        |
        v
    docker compose pull
        |
        v
    docker compose up -d
        |
        v
    Production

This prevents every code push from automatically changing production.

---

# 31. Production Deployment Flow

The complete production flow is:

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
    Manual Deploy
        |
        v
    EC2
        |
        v
    Docker Compose
        |
        v
    Node.js Container
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

# 32. Deployment Version

The deployed version is identified using:

    COMMIT_SHA

Example:

    IMAGE_TAG=COMMIT_SHA

The server pulls:

    registry.example.com/application:COMMIT_SHA

This provides an exact production version.

---

# 33. Rollback Architecture

Rollback uses an earlier image.

Conceptually:

    Current Version
          |
          v
    Problem Detected
          |
          v
    Previous Commit SHA
          |
          v
    Pull Previous Image
          |
          v
    Start Previous Image
          |
          v
    Health Check
          |
          v
    Restored Version

The image must remain available in the registry for rollback.

---

# 34. Database and Rollback

Application rollback and database rollback are different problems.

Example:

    Application Version B
          |
          v
    Database Migration
          |
          v
    Application Version B

If the application is rolled back to Version A, the database may already contain changes introduced by Version B.

Therefore:

> Application rollback does not automatically mean database rollback.

Database migrations should be designed with compatibility and rollback considerations.

---

# 35. Monitoring Architecture

Monitoring should exist at multiple levels.

External:

    Uptime Monitor
          |
          v
    HTTPS /health

Server:

    EC2
      |
      +---- CPU
      +---- Memory
      +---- Disk

Application:

    Node.js
      |
      +---- Errors
      +---- Latency
      +---- Requests
      +---- WebSocket connections

Infrastructure:

    Nginx
      |
      +---- Access Logs
      +---- Error Logs

---

# 36. Health Check

The backend exposes a health endpoint.

Generalized example:

    GET /health

A public check can be:

    curl.exe https://api.YOUR_DOMAIN/health

Expected response:

    {
      "status": "ok"
    }

The health endpoint provides a basic application availability signal.

---

# 37. Monitoring Request Flow

External monitoring:

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
           |
           v
      /health

This verifies more of the real production path than checking only localhost.

---

# 38. Logging Architecture

The application produces application logs.

Nginx produces:

- Access logs
- Error logs

Docker provides container logs.

Conceptually:

    Node.js
       |
       v
    Docker Logs

    Nginx
       |
       +---- Access Log
       |
       +---- Error Log

Future centralized logging can collect these if operational requirements justify it.

---

# 39. Resource Monitoring

EC2 resources should be monitored.

Important metrics:

    CPU
    Memory
    Disk
    Network
    Docker resources

Useful commands:

    docker stats
    free -h
    df -h
    docker system df

The purpose is to detect resource exhaustion before it becomes an outage.

---

# 40. Failure Isolation

The architecture allows failures to be investigated layer by layer.

Example:

    Public API unavailable
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
    Check Node.js
           |
           v
    Check MongoDB
           |
           v
    Check External APIs

This creates a structured troubleshooting process.

---

# 41. Security Architecture

The security model is:

    Internet
       |
       | HTTPS
       v
    Nginx
       |
       v
    Docker
       |
       v
    Node.js
       |
       +---- JWT Authentication
       |
       +---- MongoDB
       |
       +---- External APIs

Security controls include:

- HTTPS
- JWT authentication
- Runtime secrets
- Restricted network access
- GitLab protected variables
- Limited credential scopes
- Private application repository
- Docker image versioning
- No secrets in public documentation

---

# 42. Credential Separation

Different credentials have different responsibilities.

Example:

    Git Repository Credential
            |
            +---- Repository access

    Registry Credential
            |
            +---- Container Registry access

    SSH Deployment Key
            |
            +---- EC2 deployment

    Application Secrets
            |
            +---- Runtime application access

This reduces the impact of a compromised credential.

---

# 43. Public vs Private Repositories

The actual backend repository remains private.

The portfolio repository is public.

The separation is:

    PRIVATE
    Actual Application
          |
          +---- Real source code
          +---- Real configuration
          +---- Production implementation

    PUBLIC
    Portfolio Repository
          |
          +---- Generalized architecture
          +---- Sanitized commands
          +---- Examples
          +---- Lessons
          +---- Deployment process

The public repository must never become a copy of the private application.

---

# 44. Environment Separation

The architecture is designed to support multiple environments.

Conceptually:

                  Application Image
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
            DEV        TEST        PROD
             |           |           |
             v           v           v
         DEV Config  TEST Config  PROD Config
         DEV DB      TEST DB      PROD DB

The same image can be promoted between environments while configuration remains environment-specific.

---

# 45. Cost-Conscious Evolution

The initial architecture intentionally avoids unnecessary infrastructure.

Current:

    One EC2
       |
       +---- Nginx
       +---- Docker
       +---- Node.js

Database:

    MongoDB Atlas

As demand grows, the architecture can evolve.

Possible future:

    Load Balancer
          |
          +---- EC2 / Container
          |
          +---- EC2 / Container
          |
          +---- EC2 / Container

The infrastructure should evolve according to actual traffic, availability, and operational requirements.

---

# 46. Horizontal Scaling Consideration

The backend currently maintains some in-memory state, including active WebSocket connections.

This means horizontal scaling requires additional design.

Example:

    Client
       |
       v
    Load Balancer
       |
       +---- Backend 1
       |
       +---- Backend 2
       |
       +---- Backend 3

If connection/session state exists only in memory, Backend 1 and Backend 2 do not automatically share that state.

Future scaling may require:

- Shared state
- Redis
- External session management
- Message broker
- WebSocket routing strategy

This is a future architecture consideration, not part of the initial deployment.

---

# 47. Current Architecture Trade-Off

The current architecture favors:

- Simplicity
- Low infrastructure cost
- Easy deployment
- Direct server control
- Docker-based portability

The trade-offs include:

- Single EC2 runtime
- Limited redundancy
- Manual production deployment
- Basic monitoring
- More operational responsibility on the team
- Additional work required for horizontal WebSocket scaling

These trade-offs are intentional for the current stage.

---

# 48. Future Evolution

A possible future evolution is:

```text
                         Internet
                            |
                            v
                     Load Balancer
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
           EC2 #1        EC2 #2        EC2 #3
              |             |             |
              +-------------+-------------+
                            |
                            v
                    Backend Services
                       /         \
                      v           v
               MongoDB Atlas   Redis / Broker



49. Architecture Principles

The production architecture follows these principles:

1. Separate build and runtime

CI builds the artifact.

EC2 runs the artifact.

2. Immutable deployment artifacts

Use commit-SHA image tags.

3. Runtime secrets

Do not bake secrets into images.

4. Reverse proxy

Nginx is the public application boundary.

5. HTTPS

Public application traffic uses TLS.

6. Least privilege

Credentials and network access should be restricted.

7. Observable system

Health, logs, and resources should be monitored.

8. Simple infrastructure

Do not introduce infrastructure that the workload does not need.

9. Rollback capability

Keep known-good image versions available.

10. Separate application concerns

Backend, website, Electron client, database, and infrastructure have clear responsibilities.

50. Architecture Summary

The final baseline production architecture is:

Users
  |
  | HTTPS / WSS
  v
Domain
  |
  v
Nginx
  |
  v
Docker
  |
  v
Node.js Backend
  |
  +------------------+
  |                  |
  v                  v
MongoDB Atlas     External APIs

Deployment:

Developer
   |
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
EC2
   |
   v
Docker Compose
   |
   v
Production

Monitoring:

External Monitor
      |
      v
   /health
      |
      v
    Nginx
      |
      v
   Node.js

Rollback:

Production Image
      |
      v
Previous SHA Image
      |
      v
   EC2
      |
      v
   Health Check
51. Final Mental Model

The entire platform can be remembered in five layers:

Layer 1 — Client
Website
Electron
Layer 2 — Edge
DNS
HTTPS
Nginx
Layer 3 — Application
Docker
Node.js
HTTP
WebSocket
Layer 4 — Dependencies
MongoDB Atlas
External APIs
Layer 5 — Delivery & Operations
Git
GitLab CI/CD
Container Registry
EC2
Monitoring
Rollback

Full model:

CLIENT
   |
   v
EDGE
   |
   v
APPLICATION
   |
   v
DATA / EXTERNAL SERVICES

DELIVERY:
Git -> CI -> Registry -> EC2

OPERATIONS:
Monitoring -> Detection -> Investigation -> Recovery
Conclusion

The production architecture is intentionally built around a simple principle:

Build the application once,
store the immutable image,
deploy that image to controlled infrastructure,
expose it through a secure reverse proxy,
keep configuration and secrets separate,
monitor the running system,
and keep previous versions available for recovery.

This provides a practical foundation that can evolve from a small single-server deployment toward a more distributed architecture when actual scale and availability requirements demand it.


Save it. Then **`next`** → `examples/Dockerfile`.