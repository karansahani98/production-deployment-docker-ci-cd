# Phase 03 — Docker

## Status

✅ Completed

## Purpose

The goal of this phase was to package the Node.js backend into a reproducible Docker image and verify that the application works correctly inside a container.

The flow was:

```text
Node.js Application
       |
       v
Dockerfile
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
Health Check
```

---

# 1. Why Docker?

Before Docker, the application depended directly on the local machine:

```text
Windows
  |
  +-- Node.js
  |
  +-- npm dependencies
  |
  +-- MongoDB
  |
  +-- Application
```

This creates a dependency on the machine's:

- Node.js version
- npm version
- operating-system libraries
- installed dependencies
- local configuration

Docker packages the application runtime into an image:

```text
Docker Image
    |
    +-- Node.js
    +-- Application dependencies
    +-- Application
    +-- Startup configuration
```

The same image can later be pushed to a registry and run on AWS.

---

# 2. Docker Environment

The local environment used Docker Desktop on Windows.

Verify Docker:

```powershell
docker --version
```

Verify Compose:

```powershell
docker compose version
```

Check the active Docker context:

```powershell
docker context show
```

List available contexts:

```powershell
docker context ls
```

The application was run using Linux containers through Docker Desktop.

---

# 3. Repository Root

An important implementation detail:

The backend application directory itself was the repository root.

Therefore the Docker commands were executed from the backend root.

Conceptually:

```text
backend/
├── .git/
├── src/
├── shared/
├── infrastructure/
├── tests/
├── package.json
├── package-lock.json
├── Dockerfile
└── .dockerignore
```

This means:

```powershell
docker build -t my-app .
```

was executed from the backend root.

The final:

```text
.
```

means:

```text
Use the current directory as the Docker build context.
```

---

# 4. Dockerfile

The Dockerfile used for the backend followed this structure:

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

# 5. Dockerfile — Base Image

```dockerfile
FROM node:22-bookworm-slim
```

This provides:

```text
Node.js
+
Debian-based runtime
```

inside the container.

A Debian-based slim image was selected rather than Alpine because the application contains native Node.js dependencies.

One important dependency involved in this decision was:

```text
sqlite3
```

Native modules can sometimes introduce compatibility issues when moving between different Linux distributions and system libraries.

The slim Debian-based image provided a more predictable runtime for this application.

---

# 6. Dockerfile — Working Directory

```dockerfile
WORKDIR /app
```

The application runs from:

```text
/app
```

inside the container.

This gives the image a predictable internal directory structure.

---

# 7. Dockerfile — Copy Package Files

```dockerfile
COPY package*.json ./
```

This copies:

```text
package.json
package-lock.json
```

before copying the rest of the source code.

This is intentional.

Docker builds images in layers.

If the dependency files have not changed, Docker can reuse the dependency-installation layer when application source code changes.

Conceptually:

```text
package.json
package-lock.json
       |
       v
npm install
       |
       v
Dependency Layer
       |
       v
Application Source
```

---

# 8. Dockerfile — Install Dependencies

```dockerfile
RUN npm ci --omit=dev
```

The production container does not need development-only dependencies.

For example:

```text
nodemon
```

is useful during development but is not required to run the production application.

Therefore:

```text
npm ci --omit=dev
```

installs production dependencies only.

---

# 9. Dockerfile — Copy Application

```dockerfile
COPY . .
```

This copies the application source into:

```text
/app
```

inside the image.

The `.dockerignore` file determines which files are excluded from the build context.

---

# 10. Dockerfile — Expose Port

```dockerfile
EXPOSE 3000
```

The Node.js application listens on:

```text
3000
```

`EXPOSE` documents the intended container port.

It does not automatically publish the port to the host.

Port publishing happens when the container is started.

---

# 11. Dockerfile — Startup Command

```dockerfile
CMD ["npm", "start"]
```

When the container starts, Docker executes:

```text
npm start
```

Therefore the container lifecycle is tied to the Node.js application process.

If the Node.js process exits, the container stops unless a higher-level restart policy is configured.

---

# 12. `.dockerignore`

The application uses a `.dockerignore` file.

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

# 13. Why Exclude `node_modules`?

Local `node_modules` should not be copied into the image.

Instead Docker creates dependencies inside the image using:

```dockerfile
RUN npm ci --omit=dev
```

This ensures the container has dependencies installed for its own environment.

The process becomes:

```text
package-lock.json
       |
       v
npm ci
       |
       v
Container's node_modules
```

rather than:

```text
Windows node_modules
       |
       v
Linux container
```

---

# 14. Why Exclude `.env`?

The `.env` file can contain secrets.

The Docker image should not contain environment-specific credentials.

Therefore:

```text
.env
```

is excluded from the build context.

Runtime configuration should be supplied separately.

For example:

```powershell
docker run --env-file .env ...
```

The real `.env` file must never be committed to the public repository.

---

# 15. Why Exclude `.git`?

Git metadata is not required inside the production image.

Therefore:

```text
.git
```

is excluded.

This reduces the build context and prevents source-control metadata from being copied into the image.

---

# 16. Build the Image

From the application repository root:

```powershell
docker build -t my-app .
```

Explanation:

```text
docker build
      |
      +-- -t my-app
      |      |
      |      +-- image name
      |
      +-- .
             |
             +-- current directory = build context
```

---

# 17. Verify the Image

List images:

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

This confirms:

```text
Working Directory → /app
Container Port    → 3000
Startup Command   → npm start
```

---

# 18. Run the Container

For local Docker testing:

```powershell
docker run `
  --name my-app-container `
  -p 3000:3000 `
  --env-file .env `
  my-app
```

Or use a single line:

```powershell
docker run --name my-app-container -p 3000:3000 --env-file .env my-app
```

---

# 19. Port Mapping

The command contains:

```text
-p 3000:3000
```

This means:

```text
HOST PORT       CONTAINER PORT
    3000   →        3000
```

Therefore:

```text
http://localhost:3000
```

from the Windows host reaches:

```text
Docker Container :3000
```

---

# 20. Verify Container

Run:

```powershell
docker ps
```

The container should appear as running.

Then test:

```powershell
curl.exe http://localhost:3000/health
```

Expected response pattern:

```json
{
  "status": "ok"
}
```

---

# 21. Container Logs

View logs:

```powershell
docker logs my-app-container
```

Follow logs:

```powershell
docker logs -f my-app-container
```

Logs are important for identifying:

- application startup errors
- database connection errors
- missing environment variables
- runtime exceptions
- dependency failures

---

# 22. Docker Container Lifecycle Commands

Stop:

```powershell
docker stop my-app-container
```

Check all containers:

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

# 23. Major Problem Encountered — MongoDB Connectivity

One of the most important issues during Dockerization was MongoDB connectivity.

The application worked correctly when running directly on Windows.

However, after moving the Node.js application into Docker, the database connection failed when using:

```text
127.0.0.1
```

or:

```text
localhost
```

for the MongoDB host.

---

# 24. Why `localhost` Failed

When the application runs directly on Windows:

```text
Node.js
   |
   v
127.0.0.1:27017
   |
   v
MongoDB
```

This works because Node.js and MongoDB are reachable from the same host.

But inside Docker:

```text
Node.js Container
       |
       v
127.0.0.1:27017
       |
       v
The container itself
```

The container's localhost is not the Windows host.

This is a fundamental container-networking concept.

---

# 25. Solution — `host.docker.internal`

Docker Desktop provides a hostname that allows a container to reach the host machine:

```text
host.docker.internal
```

Therefore the local Docker example becomes:

```env
MONGODB_URI=mongodb://host.docker.internal:27017/example
```

The network path becomes:

```text
Docker Container
       |
       | host.docker.internal
       v
Windows Host
       |
       v
MongoDB
```

---

# 26. Local vs Docker MongoDB Configuration

## Node.js Running Directly on Windows

Example:

```env
MONGODB_URI=mongodb://localhost:27017/example
```

Network:

```text
Node.js
   |
   v
Windows
   |
   v
MongoDB
```

---

## Node.js Running Inside Docker

Example:

```env
MONGODB_URI=mongodb://host.docker.internal:27017/example
```

Network:

```text
Docker
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

---

# 27. Important Lesson

Never assume:

```text
localhost
```

means the developer's physical machine.

Always ask:

```text
Where is the process running?
```

If the process runs:

```text
Windows → localhost means Windows
```

If the process runs:

```text
Docker Container → localhost means container
```

This distinction becomes even more important when moving to:

- Docker Compose
- multiple containers
- EC2
- Kubernetes
- service-to-service communication

---

# 28. Docker Networking Debugging

List Docker networks:

```powershell
docker network ls
```

Inspect the default bridge network:

```powershell
docker network inspect bridge
```

Inspect the container:

```powershell
docker inspect my-app-container
```

### Security Warning

`docker inspect` can contain environment variables.

Never publish its raw output if it contains:

```text
passwords
API keys
tokens
database credentials
JWT secrets
```

---

# 29. Port Troubleshooting

If port 3000 is already in use:

```powershell
netstat -ano | findstr :3000
```

Then identify the process:

```powershell
tasklist | findstr <PID>
```

The `<PID>` is only a local placeholder.

---

# 30. Container Startup Failure

If the container exits:

```powershell
docker ps -a
```

Then:

```powershell
docker logs my-app-container
```

This usually gives the first indication of the problem.

Typical categories:

```text
Application error
Database connection error
Missing environment variable
Dependency problem
Port problem
Runtime exception
```

---

# 31. Rebuild the Image

After changing the Dockerfile:

```powershell
docker build -t my-app .
```

If the cache needs to be bypassed:

```powershell
docker build --no-cache -t my-app .
```

Use `--no-cache` only when necessary.

---

# 32. Inspect Image Configuration

The following command was useful during the implementation:

```powershell
docker image inspect my-app --format "{{.Config.WorkingDir}} | {{.Config.ExposedPorts}} | {{.Config.Cmd}}"
```

It verifies the important runtime configuration without requiring us to inspect the entire image metadata.

---

# 33. Production Image Principle

The Docker image should contain:

```text
Application
+
Runtime dependencies
+
Node.js runtime
```

It should not contain:

```text
Environment secrets
+
Developer machine dependencies
+
Git metadata
+
Temporary files
```

This produces a cleaner and safer deployment artifact.

---

# 34. Build Once, Run Everywhere

Dockerization gives us the foundation for:

```text
Source Code
     |
     v
Docker Build
     |
     v
Versioned Image
     |
     +----------+
     |          |
     v          v
   DEV        TEST
                |
                v
              PROD
```

The image can later be stored in the container registry.

The production server does not need to build the application from source.

It can simply pull the required image.

---

# 35. Why This Matters for CI/CD

Without Docker:

```text
GitLab
   |
   v
EC2
   |
   +-- install Node.js
   +-- install dependencies
   +-- configure runtime
   +-- copy source
   +-- start application
```

With Docker:

```text
GitLab
   |
   v
Docker Build
   |
   v
Container Registry
   |
   v
EC2
   |
   v
docker pull
   |
   v
docker run
```

This significantly simplifies deployment consistency.

---

# 36. Verification Checklist

```text
[✓] Docker installed
[✓] Docker context verified
[✓] Dockerfile created
[✓] .dockerignore created
[✓] Production dependencies installed in image
[✓] Docker image built
[✓] Image inspected
[✓] Container started
[✓] Port 3000 mapped
[✓] Container logs checked
[✓] Health endpoint tested
[✓] MongoDB connectivity issue identified
[✓] host.docker.internal solution applied
[✓] Docker application verified
```

---

# 37. Important Engineering Decisions

## Decision 1 — Node.js Debian Slim

Use:

```dockerfile
FROM node:22-bookworm-slim
```

because the application has native dependencies.

---

## Decision 2 — `npm ci --omit=dev`

Production image installation:

```dockerfile
RUN npm ci --omit=dev
```

Development dependencies are not required in the production runtime.

---

## Decision 3 — Runtime Environment Configuration

The image does not contain `.env`.

Environment-specific values are supplied at runtime.

---

## Decision 4 — Docker Build From Repository Root

The application directory itself is the repository root.

Therefore:

```powershell
docker build -t my-app .
```

is run from that directory.

---

## Decision 5 — Explicit Health Verification

The application must be tested through:

```powershell
curl.exe http://localhost:3000/health
```

after starting the container.

A running container alone does not prove that the application is healthy.

---

# 38. Lessons Learned

### Lesson 1 — Containers are isolated environments

A container should not be treated as another terminal on the developer's Windows machine.

---

### Lesson 2 — `localhost` depends on execution context

The meaning of localhost changes depending on where the process is running.

---

### Lesson 3 — Docker images should be reproducible

Dependencies should be installed inside the image rather than copied from the developer machine.

---

### Lesson 4 — Secrets should be runtime configuration

Do not bake secrets into Docker images.

---

### Lesson 5 — Verify the actual image

`docker image inspect` is useful for confirming:

```text
Working directory
Exposed ports
Startup command
```

---

# 39. Phase Result

The backend was successfully converted from:

```text
Local Node.js Application
```

into:

```text
Docker Image
```

and then:

```text
Docker Container
```

The container was verified through the health endpoint.

The MongoDB networking problem was identified and solved for Docker Desktop local development.

The resulting foundation is:

```text
Application
     |
     v
Docker Image
     |
     v
Docker Container
     |
     v
Health Check
```

---

# 40. Phase Completion

```text
✅ PHASE 03 — DOCKER COMPLETED
```

---

# 41. Next Phase

➡️ **Phase 04 — Container Registry**

The next phase will move the Docker image from the local machine into a centralized container registry:

```text
Local Docker Image
       |
       v
Docker Tag
       |
       v
Container Registry
       |
       v
Registry Image
```

We will document:

- Registry setup
- Docker login
- Image tagging
- Image push
- Commit-SHA tagging
- Registry verification
- Credential separation
- Security considerations
- Problems encountered
- Lessons learned

All real registry credentials and private infrastructure information will remain excluded.