# Phase 01 — Local Development

## Status

✅ Completed

## Purpose

Before introducing GitLab, Docker Registry, AWS, Nginx, HTTPS, and CI/CD, we first verified that the backend application worked correctly in the local development environment.

The basic principle was:

```text
Application
    ↓
Local Environment
    ↓
Database
    ↓
API
    ↓
Health Check
```

Only after this baseline was working did we move toward containerization and production infrastructure.

---

# 1. Application Context

The application used during this deployment journey is a Node.js backend.

The actual application is maintained in a **private repository**.

This public portfolio repository does not contain the application's real source code.

It documents the engineering process used to take a backend application from local development to production.

The backend contains:

- HTTP APIs
- Authentication
- WebSocket communication
- Database connectivity
- External service integrations

The application listens on:

```text
Port 3000
```

---

# 2. Local Development Environment

The development environment used during this phase was:

```text
Operating System : Windows
Shell            : PowerShell
Docker            : Docker Desktop
Container Runtime : Linux containers
Database          : MongoDB
Backend           : Node.js
```

The actual application was maintained under a local development directory similar to:

```text
C:\path\to\application
```

The exact real path is intentionally not documented here.

---

# 3. Application Repository Structure

The backend repository followed this general structure:

```text
application/
│
├── infrastructure/
├── shared/
├── src/
├── test/
├── tests/
│
├── .dockerignore
├── .gitignore
├── Dockerfile
├── package.json
├── package-lock.json
└── server.js
```

The important point is that the backend directory itself was treated as the application/repository root.

This became important later when GitLab CI/CD was configured.

---

# 4. Node.js Project

The application uses Node.js.

The project contains:

```text
package.json
package-lock.json
```

The package scripts follow this general pattern:

```json
{
  "scripts": {
    "dev": "nodemon src/index.js",
    "start": "node src/index.js"
  }
}
```

Development:

```powershell
npm run dev
```

Production-style startup:

```powershell
npm start
```

---

# 5. Verify Node.js

Check the installed Node.js version:

```powershell
node --version
```

Check npm:

```powershell
npm --version
```

The exact patch version depends on the developer machine.

For a production deployment project, the important requirement is that the runtime version is explicitly known and consistent with the Docker image.

---

# 6. Install Dependencies

From the application repository root:

```powershell
npm ci
```

Why `npm ci`?

The repository contains:

```text
package-lock.json
```

`npm ci` uses the lock file to install the dependency tree reproducibly.

This becomes particularly important later when Docker builds the production image.

---

# 7. Start the Application

For development:

```powershell
npm run dev
```

For a production-style local startup:

```powershell
npm start
```

The backend listens on:

```text
localhost:3000
```

---

# 8. Health Endpoint

The backend provides a health endpoint:

```text
GET /health
```

Test it locally:

```powershell
curl.exe http://localhost:3000/health
```

Expected response pattern:

```json
{
  "status": "ok"
}
```

The actual response can contain additional runtime information such as uptime and timestamp.

---

# 9. Why the Health Endpoint Matters

The health endpoint was not only useful for local development.

It later becomes useful for:

```text
Local verification
      ↓
Docker verification
      ↓
Production verification
      ↓
CI/CD deployment verification
      ↓
Rollback verification
      ↓
Monitoring
```

This is why a simple health endpoint is valuable in a production application.

---

# 10. Environment Configuration

The application uses environment variables for configuration.

Typical examples:

```env
PORT=3000
MONGODB_URI=YOUR_MONGODB_URI
JWT_SECRET=YOUR_SECRET
```

The actual values are private.

### Public documentation rule

Never put real values into this repository.

Use placeholders:

```env
PORT=3000
MONGODB_URI=YOUR_MONGODB_URI
JWT_SECRET=YOUR_SECRET
```

---

# 11. `.env` Security

The local `.env` file contains environment-specific configuration.

It must not be committed to Git.

The public project uses:

```gitignore
.env
```

The same principle applies to:

```text
API keys
Database passwords
JWT secrets
Encryption keys
SSH keys
Registry credentials
Production credentials
```

---

# 12. Local MongoDB

During local development, MongoDB was available outside the Docker container.

When the Node.js application runs directly on Windows, a local MongoDB connection can use:

```env
MONGODB_URI=mongodb://127.0.0.1:27017/example
```

or:

```env
MONGODB_URI=mongodb://localhost:27017/example
```

These are examples only.

The important concept is:

```text
Node.js
   |
   v
Windows Host
   |
   v
MongoDB :27017
```

---

# 13. Verify Local Database Connectivity

Start the backend:

```powershell
npm start
```

Then inspect the application logs.

The application should establish its MongoDB connection before serving normal database-dependent requests.

If the connection fails, check:

```text
MongoDB is running
MongoDB port is correct
Connection string is correct
Environment variable is loaded
Network access is available
```

Never publish the actual connection string if it contains credentials.

---

# 14. Docker Preparation

After verifying the application locally, we prepared it to run inside Docker.

The first important file is:

```text
Dockerfile
```

The generalized Dockerfile used during the implementation was:

```dockerfile
FROM node:22-bookworm-slim

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

---

# 15. Dockerfile Explanation

## Base Image

```dockerfile
FROM node:22-bookworm-slim
```

This provides the Node.js runtime inside the container.

A Debian-based slim image was selected instead of Alpine because the application uses native Node.js dependencies, including SQLite-related packages.

The goal was to avoid unnecessary native-module compatibility problems.

---

## Working Directory

```dockerfile
WORKDIR /app
```

The application runs from:

```text
/app
```

inside the container.

---

## Copy Package Files First

```dockerfile
COPY package*.json ./
```

This copies:

```text
package.json
package-lock.json
```

before copying the application source.

This allows Docker to reuse the dependency-installation layer when application source files change but dependencies do not.

---

## Install Production Dependencies

```dockerfile
RUN npm ci --omit=dev
```

Production images do not need development dependencies such as:

```text
nodemon
```

Therefore:

```text
npm ci --omit=dev
```

is used for the production container.

---

## Copy Application

```dockerfile
COPY . .
```

This copies the application into:

```text
/app
```

The `.dockerignore` file controls what is excluded from the build context.

---

## Expose Application Port

```dockerfile
EXPOSE 3000
```

This documents the application's listening port.

It does not by itself publish the port to the host.

Port publishing happens when the container is run.

---

## Startup Command

```dockerfile
CMD ["npm", "start"]
```

The container starts the backend using:

```text
npm start
```

---

# 16. `.dockerignore`

The application also uses `.dockerignore`.

General configuration:

```text
node_modules
.env
npm-debug.log
.git
.gitignore
Dockerfile
.dockerignore
```

The most important exclusions are:

```text
node_modules
.env
.git
```

---

# 17. Why `.env` Must Not Be in the Image

The Docker image should contain:

```text
Application
+
Runtime dependencies
```

It should not contain:

```text
Production secrets
```

Therefore:

```text
.env
```

is excluded.

Environment-specific values should be supplied at runtime.

---

# 18. Build the Docker Image

From the application repository root:

```powershell
docker build -t my-app .
```

The final `.` means:

```text
Use the current directory as the Docker build context.
```

Important:

The application repository root is already the backend root.

Therefore, we do not need:

```powershell
cd server
```

when the current working directory is already the repository root.

This distinction later prevented a GitLab CI path problem.

---

# 19. Verify Docker Image

List Docker images:

```powershell
docker images
```

Inspect the image:

```powershell
docker image inspect my-app
```

A focused inspection:

```powershell
docker image inspect my-app --format "{{.Config.WorkingDir}} | {{.Config.ExposedPorts}} | {{.Config.Cmd}}"
```

Expected pattern:

```text
/app | map[3000/tcp:{}] | [npm start]
```

This verifies:

```text
Working directory → /app
Port              → 3000
Startup command   → npm start
```

---

# 20. Run the Docker Container

For local testing:

```powershell
docker run `
  --name my-app-container `
  -p 3000:3000 `
  --env-file .env `
  my-app
```

Or as a single command:

```powershell
docker run --name my-app-container -p 3000:3000 --env-file .env my-app
```

The port mapping:

```text
3000:3000
```

means:

```text
Windows Host :3000
       |
       v
Container :3000
```

---

# 21. Verify Running Container

Run:

```powershell
docker ps
```

The application container should be running.

Then test:

```powershell
curl.exe http://localhost:3000/health
```

The request path is:

```text
PowerShell
    |
    v
localhost:3000
    |
    v
Docker Container
    |
    v
Node.js
    |
    v
/health
```

---

# 22. Docker Logs

View container logs:

```powershell
docker logs my-app-container
```

Follow logs:

```powershell
docker logs -f my-app-container
```

These logs are especially useful when:

- Application startup fails
- Database connection fails
- Environment variables are missing
- The process crashes
- A dependency fails to initialize

---

# 23. Docker Container Lifecycle

Stop:

```powershell
docker stop my-app-container
```

List all containers:

```powershell
docker ps -a
```

Remove:

```powershell
docker rm my-app-container
```

List images:

```powershell
docker images
```

---

# 24. Important Problem — MongoDB Connection From Docker

This was one of the important troubleshooting points during implementation.

The application worked when running directly on Windows.

However, when the application was moved into Docker, the same:

```env
MONGODB_URI=mongodb://127.0.0.1:27017/example
```

did not refer to the Windows host.

---

# 25. Why `localhost` Failed

Inside a container:

```text
127.0.0.1
```

means:

```text
the container itself
```

Therefore:

```text
Docker Container
      |
      | 127.0.0.1:27017
      v
Docker Container
```

It does not mean:

```text
Windows Host
```

---

# 26. Correct Docker Desktop Host Address

For Docker Desktop on Windows, the host can be reached using:

```text
host.docker.internal
```

Therefore the local Docker example becomes:

```env
MONGODB_URI=mongodb://host.docker.internal:27017/example
```

The resulting network path is:

```text
Docker Container
       |
       | host.docker.internal
       v
Windows Host
       |
       v
MongoDB :27017
```

This solved the local Docker-to-host database connectivity issue.

---

# 27. Two Different Networking Situations

## Application Runs Directly on Windows

```text
Node.js
   |
   v
localhost
   |
   v
MongoDB
```

Example:

```env
MONGODB_URI=mongodb://localhost:27017/example
```

---

## Application Runs Inside Docker

```text
Node.js Container
       |
       v
host.docker.internal
       |
       v
Windows Host
       |
       v
MongoDB
```

Example:

```env
MONGODB_URI=mongodb://host.docker.internal:27017/example
```

The configuration depends on where the application process is running.

---

# 28. Docker Network Debugging

List Docker networks:

```powershell
docker network ls
```

Inspect the default bridge network:

```powershell
docker network inspect bridge
```

Inspect a container:

```powershell
docker inspect my-app-container
```

### Security Warning

`docker inspect` can contain environment variables.

Do not publish its output without checking it for secrets.

---

# 29. Port Troubleshooting

If port `3000` is already being used:

```powershell
netstat -ano | findstr :3000
```

Then identify the process:

```powershell
tasklist | findstr <PID>
```

Use the actual PID only on the local machine.

---

# 30. Container Exited Unexpectedly

Check:

```powershell
docker ps -a
```

Then:

```powershell
docker logs my-app-container
```

The logs usually reveal whether the problem is related to:

```text
Application startup
Database connection
Environment variables
Missing dependency
Port configuration
Runtime exception
```

---

# 31. Rebuild Docker Image

After changing the Dockerfile:

```powershell
docker build -t my-app .
```

If Docker cache needs to be bypassed:

```powershell
docker build --no-cache -t my-app .
```

`--no-cache` should be used only when required because it makes the build slower.

---

# 32. Local Development Verification

Before continuing to the next phase, the following workflow was verified:

```text
Node.js Application
       |
       v
npm ci
       |
       v
npm start
       |
       v
localhost:3000
       |
       v
/health
```

Then:

```text
Node.js Application
       |
       v
Docker Build
       |
       v
Docker Image
       |
       v
Docker Container
       |
       v
localhost:3000
       |
       v
/health
```

Database connectivity was also verified for the Docker environment.

---

# 33. Verification Checklist

```text
[✓] Node.js installed
[✓] npm verified
[✓] Dependencies installed
[✓] Application started
[✓] Health endpoint verified
[✓] Local MongoDB connectivity verified
[✓] Docker installed
[✓] Docker image built
[✓] Docker image inspected
[✓] Docker container started
[✓] Container logs checked
[✓] API tested through Docker
[✓] Docker-to-host MongoDB issue identified
[✓] host.docker.internal solution applied
```

---

# 34. Important Engineering Decisions

## Decision 1 — Separate Local and Production Configuration

Local configuration and production configuration are different.

Secrets are supplied through environment configuration rather than committed into source control.

---

## Decision 2 — Production Docker Image Excludes Dev Dependencies

The image uses:

```bash
npm ci --omit=dev
```

This keeps the production image focused on runtime dependencies.

---

## Decision 3 — Debian Slim Instead of Alpine

The application uses native dependencies.

A Debian-based slim Node.js image was selected to reduce native dependency compatibility issues.

---

## Decision 4 — Application Repository Root

The backend directory itself is treated as the repository root.

This means:

```text
Git root      → backend root
Docker context → backend root
CI working dir → backend root
```

This becomes important during CI/CD configuration.

---

## Decision 5 — Health Endpoint From the Beginning

The `/health` endpoint is used as a common verification mechanism.

Future uses include:

```text
Deployment health check
Rollback validation
Monitoring
Load balancer health check
```

---

# 35. Lessons Learned

### 1. Container localhost is not the host machine

This is one of the most important Docker networking concepts.

```text
localhost
```

always needs to be understood in the context of the process/container where it is being used.

---

### 2. Verify one layer before adding another

The deployment journey follows:

```text
Local Application
       ↓
Docker
       ↓
Registry
       ↓
AWS
       ↓
Nginx
       ↓
HTTPS
       ↓
CI/CD
```

This makes troubleshooting much easier.

---

### 3. Secrets belong outside the image and repository

The application image should not contain:

```text
.env
API keys
Passwords
JWT secrets
Database credentials
```

---

### 4. Repository structure matters

Knowing the actual repository root prevents mistakes in:

```text
Docker builds
Git commands
CI/CD scripts
Deployment scripts
```

---

# 36. Phase Completion

Phase 01 is complete when:

```text
Local application works
        +
Docker image builds
        +
Docker container works
        +
Database connectivity works
        +
Health endpoint works
```

Current status:

```text
✅ PHASE 01 COMPLETED
```

---

# 37. Next Phase

➡️ **Phase 02 — Git Workflow**

The next phase documents the actual process of:

```text
Git initialization
       ↓
.gitignore
       ↓
main branch
       ↓
GitLab repository
       ↓
Remote configuration
       ↓
Authentication
       ↓
Credential troubleshooting
       ↓
First push
       ↓
Verification
```

All real repository information and credentials will remain private.