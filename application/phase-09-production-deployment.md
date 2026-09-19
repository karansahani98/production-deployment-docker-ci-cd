# Phase 09 — Production Deployment

## 1. Goal

The goal of this phase is to document the complete production deployment workflow from source-code change to a running production application.

By this point, the following components are already configured:

```text
GitLab Repository
       |
       v
GitLab CI/CD
       |
       v
Docker Image
       |
       v
Container Registry
       |
       v
AWS EC2
       |
       v
Docker Container
       |
       v
Nginx
       |
       v
HTTPS Domain
       |
       v
Node.js Backend
       |
       v
MongoDB Atlas
```

This phase connects all of these pieces into one repeatable deployment process.

---

# 2. Final Production Flow

The complete deployment flow is:

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
    +--> Build Docker Image
    |
    +--> Tag with Commit SHA
    |
    +--> Push to Container Registry
    |
    v
Manual Production Deployment
    |
    | SSH
    v
AWS EC2
    |
    v
Docker Compose
    |
    +--> Pull exact image
    |
    +--> Start/recreate container
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

---

# 3. Deployment Philosophy

The deployment process follows three important principles:

```text
1. Source code is stored in Git
2. Docker image is the deployment artifact
3. Production runs a specific image version
```

The production server should not build the application from source.

Instead:

```text
Git
 |
 v
CI
 |
 v
Docker Image
 |
 v
Registry
 |
 v
Production
```

---

# 4. Source Code to Production

A code change starts with the developer.

Example:

```text
Developer changes code
        |
        v
git status
        |
        v
git add
        |
        v
git commit
        |
        v
git push
```

The push triggers GitLab CI/CD.

---

# 5. Git Workflow

Before pushing:

```powershell
git status
```

Review changed files.

Then:

```powershell
git add .
```

Create a commit:

```powershell
git commit -m "Update backend functionality"
```

Push:

```powershell
git push
```

The exact commit message will depend on the change.

The important part is that every deployment begins with a versioned Git commit.

---

# 6. GitLab Pipeline Trigger

After:

```bash
git push
```

GitLab creates a pipeline.

The pipeline starts the build process.

Conceptually:

```text
git push
   |
   v
GitLab
   |
   v
Pipeline
   |
   v
Build
```

The pipeline should be visible in the GitLab CI/CD interface.

---

# 7. Build Stage

The build stage:

```text
Source Code
    |
    v
Docker Build
    |
    v
Docker Image
```

The Docker image is built using the application's Dockerfile.

Conceptually:

```bash
docker build -t IMAGE_NAME:IMAGE_TAG .
```

The production image is not manually built on EC2.

---

# 8. Image Version

The image tag is based on:

```text
CI_COMMIT_SHA
```

Therefore:

```text
Git Commit
    |
    v
CI_COMMIT_SHA
    |
    v
Docker Image Tag
```

Example:

```text
IMAGE_NAME:abc123
```

The actual image name and commit values are intentionally not included in this public documentation.

---

# 9. Why the Commit SHA Matters

Suppose three deployments happen:

```text
Commit A
Commit B
Commit C
```

The registry contains:

```text
image:A
image:B
image:C
```

Production can explicitly run:

```text
image:C
```

If a rollback is required:

```text
image:B
```

can be selected.

This gives the deployment process a clear version history.

---

# 10. Registry Push

After the Docker image is built:

```bash
docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
```

The image is stored in the GitLab Container Registry.

The flow is:

```text
GitLab CI
    |
    v
Docker Build
    |
    v
Versioned Image
    |
    v
Container Registry
```

At this point the image is ready for deployment.

---

# 11. Production Deployment Is Manual

The production deployment job is configured as manual.

Conceptually:

```text
Build
  |
  v
Image pushed
  |
  v
WAIT
  |
  v
Manual Deploy
```

This means a normal code push does not automatically change production.

The developer can review the pipeline and then trigger the production deployment.

---

# 12. Why Separate Build and Deploy?

This separation provides a useful release boundary.

```text
BUILD
  |
  v
Artifact Ready
  |
  v
RELEASE DECISION
  |
  v
DEPLOY
```

The build answers:

> Can we produce the application artifact?

The deployment answers:

> Should this artifact be released to production now?

Keeping these concerns separate makes the process easier to control.

---

# 13. Deployment Job

The deployment job connects to EC2 using SSH.

Conceptually:

```bash
ssh "$EC2_USER@$EC2_HOST"
```

The actual infrastructure values are stored in GitLab CI/CD Variables.

Use placeholders in documentation:

```text
YOUR_EC2_USER
YOUR_EC2_IP
YOUR_SSH_PRIVATE_KEY
```

Never commit real values.

---

# 14. SSH Preparation

The deployment job installs the SSH client:

```bash
apk add --no-cache openssh-client
```

Then creates:

```bash
mkdir -p ~/.ssh
```

and:

```bash
chmod 700 ~/.ssh
```

The private key is written with restrictive permissions:

```bash
chmod 600 ~/.ssh/id_rsa
```

This allows the CI runner to authenticate to EC2.

---

# 15. SSH Host Verification

The deployment job prepares:

```bash
ssh-keyscan -H "$EC2_HOST" >> ~/.ssh/known_hosts
```

This prevents the deployment from stopping at an interactive SSH confirmation prompt.

For a stronger production security implementation, the expected host key should be managed explicitly rather than blindly trusting a newly scanned key.

This remains a security-hardening consideration.

---

# 16. EC2 Deployment Directory

The deployment job enters the application deployment directory.

Example:

```bash
cd /home/YOUR_EC2_USER/YOUR_APPLICATION/backend
```

The actual production path is intentionally not included in the public repository.

The directory contains deployment configuration such as:

```text
backend/
    |
    +--> docker-compose.yml
    |
    +--> .env
```

---

# 17. Pull Exact Image

The first deployment operation is:

```bash
IMAGE_TAG=$CI_COMMIT_SHA docker compose pull
```

This tells Docker Compose to pull the exact image version produced by the current Git commit.

Conceptually:

```text
CI_COMMIT_SHA
      |
      v
Docker Image
      |
      v
Registry
      |
      | pull
      v
EC2
```

---

# 18. Why Pull Before Start?

The production server should obtain the intended image before starting the new container.

The sequence is:

```text
Pull
 |
 v
Verify image availability
 |
 v
Start container
```

If the image cannot be pulled, the deployment should not proceed to start the new version.

---

# 19. Start the New Version

After pulling:

```bash
IMAGE_TAG=$CI_COMMIT_SHA docker compose up -d
```

Docker Compose uses the image specified by:

```text
IMAGE_TAG
```

and starts the backend.

The deployment flow is:

```text
Registry
    |
    v
docker compose pull
    |
    v
Local EC2 image
    |
    v
docker compose up -d
    |
    v
Running container
```

---

# 20. Verify Container Status

After starting the application:

```bash
IMAGE_TAG=$CI_COMMIT_SHA docker compose ps
```

This confirms the container state.

The expected state is:

```text
running
```

If the container is not running, investigate before considering the deployment successful.

---

# 21. Check Container Logs

If required:

```bash
docker compose logs backend
```

For live logs:

```bash
docker compose logs -f backend
```

Look for:

```text
Application startup
Database connection
Server listening
Unexpected exceptions
Configuration errors
```

Never publish logs containing production secrets.

---

# 22. Verify the Application

After the container starts, verify the backend health endpoint.

From the EC2 server:

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

---

# 23. Verify Nginx

Next verify the public reverse proxy.

Example:

```bash
curl http://api.example.com/health
```

Then verify HTTPS:

```bash
curl https://api.example.com/health
```

The exact production domain is intentionally replaced with:

```text
api.example.com
```

in this documentation.

---

# 24. Production Verification Layers

A production deployment should be verified from multiple layers.

## Layer 1 — Container

```bash
docker compose ps
```

## Layer 2 — Application

```bash
curl http://127.0.0.1:3000/health
```

## Layer 3 — Nginx

```bash
curl http://api.example.com/health
```

## Layer 4 — HTTPS

```bash
curl https://api.example.com/health
```

## Layer 5 — Real application flow

Verify important API functionality such as:

```text
Authentication
API requests
Database access
WebSocket connection
External service integration
```

---

# 25. Complete Deployment Sequence

The complete production deployment can be summarized as:

```text
1. Developer changes code
        |
        v
2. git commit
        |
        v
3. git push
        |
        v
4. GitLab pipeline starts
        |
        v
5. Docker image builds
        |
        v
6. Image tagged with CI_COMMIT_SHA
        |
        v
7. Image pushed to registry
        |
        v
8. Production deployment manually triggered
        |
        v
9. GitLab runner connects to EC2
        |
        v
10. docker compose pull
        |
        v
11. docker compose up -d
        |
        v
12. docker compose ps
        |
        v
13. Health check
        |
        v
14. Production verification
```

---

# 26. Production Deployment Command Pattern

The deployment command pattern is:

```bash
IMAGE_TAG=$CI_COMMIT_SHA docker compose pull
IMAGE_TAG=$CI_COMMIT_SHA docker compose up -d
IMAGE_TAG=$CI_COMMIT_SHA docker compose ps
```

The same image tag is used throughout.

This prevents an accidental mismatch between:

```text
pull version
```

and:

```text
running version
```

---

# 27. Why Not Use latest?

Avoid:

```text
latest
```

for controlled production deployments.

With:

```text
latest
```

you can lose the explicit relationship between:

```text
Git Commit
```

and:

```text
Production Image
```

Instead:

```text
CI_COMMIT_SHA
```

provides a deterministic deployment reference.

---

# 28. Example Deployment History

Imagine:

```text
Commit A
Commit B
Commit C
Commit D
```

The registry contains:

```text
image:A
image:B
image:C
image:D
```

Production history could then be:

```text
Deployment 1 -> image:A
Deployment 2 -> image:B
Deployment 3 -> image:C
Deployment 4 -> image:D
```

If version D has a problem, rollback can target:

```text
image:C
```

The rollback procedure is documented in Phase 10.

---

# 29. Health Check Strategy

The application already provides a health endpoint:

```text
/health
```

The deployment process can use this endpoint to verify the application.

Current verification:

```text
Deployment
    |
    v
Container running
    |
    v
HTTPS /health
    |
    v
status = ok
```

A future improvement is to make the CI deployment job automatically perform this health check and fail the deployment if the health check does not succeed.

---

# 30. Automatic Rollback

Automatic rollback is not yet part of the current deployment implementation.

The future architecture can be:

```text
Deploy
   |
   v
Health Check
   |
   +----> PASS
   |        |
   |        v
   |     Success
   |
   +----> FAIL
            |
            v
       Rollback
            |
            v
       Previous Image
```

This will be designed and tested separately.

Do not add automatic rollback until the rollback process itself has been tested safely.

---

# 31. Deployment Failure Scenarios

A deployment can fail at several points.

## Build failure

```text
Source
  |
  v
Docker Build
  |
  X
Failure
```

Production is not changed.

---

## Registry push failure

```text
Build
  |
  v
Registry Push
  |
  X
Failure
```

Production is not changed.

---

## SSH failure

```text
Deploy
  |
  v
SSH
  |
  X
Failure
```

The deployment cannot reach EC2.

---

## Image pull failure

```text
EC2
  |
  v
docker compose pull
  |
  X
Failure
```

The new image is not available locally.

---

## Container startup failure

```text
Image
  |
  v
docker compose up
  |
  X
Container fails
```

Logs must be inspected.

---

## Application health failure

```text
Container
   |
   v
Health Check
   |
   X
Failure
```

This should eventually trigger an automated rollback mechanism.

---

# 32. Deployment Safety

A production deployment should not be considered successful simply because:

```bash
docker compose up -d
```

returns successfully.

The container can start while the application itself is unhealthy.

Therefore:

```text
Container started
       !=
Application healthy
```

A proper deployment verification should include:

```text
Container status
+
Application health
+
Public HTTPS endpoint
+
Critical application flow
```

---

# 33. Application Health vs Infrastructure Health

These are different.

Infrastructure health:

```text
Docker container = running
```

Application health:

```text
/health = OK
```

Production functionality:

```text
Login works
API works
Database works
WebSocket works
```

A stronger deployment process verifies all three levels.

---

# 34. Database Considerations

The application uses MongoDB Atlas.

During deployment, the container must continue using the correct production database configuration.

The deployment should not accidentally point production to:

```text
Local MongoDB
Development MongoDB
Test MongoDB
```

Environment configuration must remain separate.

Conceptually:

```text
DEV
 |
 +--> DEV database

TEST
 |
 +--> TEST database

PROD
 |
 +--> Production MongoDB
```

The actual database URI must never be stored in public documentation.

---

# 35. External Services

The backend also depends on external services.

Examples include:

```text
AI APIs
Speech/Audio APIs
Other third-party services
```

A deployment can therefore succeed at the container level while an external dependency is unavailable.

This is another reason why application-level health checks and functional verification matter.

---

# 36. WebSocket Verification

Because the application uses WebSockets, deployment verification should include the WebSocket path.

Conceptually:

```text
Client
   |
   | WSS
   v
Nginx
   |
   | Upgrade
   v
Node.js WebSocket
```

The public endpoint follows the secure WebSocket pattern:

```text
wss://api.example.com/stream
```

The exact path depends on the application's WebSocket implementation.

---

# 37. Production Deployment Checklist

Before deployment:

```text
[ ] Code reviewed
[ ] Git status checked
[ ] Commit created
[ ] Push completed
[ ] CI build passed
[ ] Docker image pushed
[ ] Correct commit SHA identified
```

During deployment:

```text
[ ] Manual production deployment triggered
[ ] SSH connection succeeds
[ ] Exact image pulled
[ ] Container recreated/started
[ ] Container status checked
```

After deployment:

```text
[ ] Application logs checked
[ ] Local health check passed
[ ] HTTPS health check passed
[ ] Authentication verified
[ ] Database access verified
[ ] WebSocket verified
```

---

# 38. Rollback Preparation

Before considering a deployment complete, record the deployed image version.

For example:

```text
Current Image:
YOUR_REGISTRY_IMAGE:YOUR_COMMIT_SHA
```

The actual value should be obtained from the deployment itself and should not be hardcoded into public documentation.

This makes it easier to identify the current production version.

---

# 39. Deployment Audit Trail

One benefit of using GitLab CI/CD is that the deployment history can be connected to:

```text
Git commit
    |
    v
Pipeline
    |
    v
Docker image
    |
    v
Production deployment
```

This creates an audit trail.

When debugging a production issue, you can ask:

```text
Which commit was deployed?
Which image was built?
Which image is running?
When was it deployed?
```

This is much easier when deployments are versioned.

---

# 40. Important Security Rules

Never place these in the public repository:

```text
Production IP
Private SSH key
MongoDB URI
Database password
JWT secret
Encryption secret
API keys
Cloud credentials
Registry credentials
```

Use placeholders:

```text
YOUR_EC2_IP
YOUR_EC2_USER
YOUR_DOMAIN
YOUR_REGISTRY_IMAGE
YOUR_SECRET
```

---

# 41. Production Secret Handling

The application requires secrets such as:

```text
MONGODB_URI
JWT_SECRET
API credentials
Encryption keys
```

These should not be included in:

```text
Dockerfile
.gitlab-ci.yml
README.md
Git history
Public GitHub documentation
```

The current implementation keeps production environment configuration on the server.

Future work should move toward dedicated production secret management.

---

# 42. Important Deployment Decisions

## Decision 1 — Manual production deployment

Production is not deployed automatically after every push.

```text
Push
 |
 v
Build
 |
 v
Registry
 |
 v
Manual Deploy
```

---

## Decision 2 — Immutable image reference

The deployment uses:

```text
CI_COMMIT_SHA
```

instead of:

```text
latest
```

---

## Decision 3 — EC2 does not build source code

EC2 only pulls and runs the Docker image.

```text
CI
 |
 v
Build
 |
 v
Registry
 |
 v
EC2
```

---

## Decision 4 — Verify after deployment

Deployment is not considered complete until the running application has been checked.

---

## Decision 5 — Keep rollback separate

Rollback should be a deliberate and tested process rather than an untested automatic mechanism.

---

# 43. Lessons Learned

### Lesson 1

A successful Docker deployment is not automatically a successful production release.

### Lesson 2

Always verify the application after deployment.

### Lesson 3

Use immutable image references for production.

### Lesson 4

Separate build and release decisions.

### Lesson 5

Keep production secrets outside source control.

### Lesson 6

Production deployment should be reproducible.

### Lesson 7

Health checks should eventually become part of the deployment pipeline.

### Lesson 8

Rollback should be tested before automating it.

### Lesson 9

Application-level verification is more important than only checking container status.

### Lesson 10

A deployment should be traceable back to a specific Git commit.

---

# 44. Current Production Deployment Architecture

The complete system is:

```text
                         INTERNET
                            |
                            v
                     API DOMAIN / DNS
                            |
                            v
                      NGINX :443
                            |
                            v
                    DOCKER CONTAINER
                            |
                            v
                     NODE.JS BACKEND
                            |
                            v
                      MONGODB ATLAS
```

Deployment path:

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
    +--> Docker Build
    |
    +--> Commit SHA Tag
    |
    +--> Container Registry
    |
    v
Manual Deploy
    |
    | SSH
    v
AWS EC2
    |
    v
Docker Compose
    |
    v
Production Container
```

---

# 45. Verification Checklist

Before considering this phase complete:

- [x] Source code pushed to Git
- [x] GitLab pipeline triggered
- [x] Docker image built
- [x] Image tagged using commit SHA
- [x] Image pushed to Container Registry
- [x] Production deployment configured as manual
- [x] GitLab runner connects to EC2
- [x] Exact image pulled
- [x] Docker Compose starts application
- [x] Container status verified
- [x] Application logs verified
- [x] Local health endpoint verified
- [x] Nginx endpoint verified
- [x] HTTPS endpoint verified
- [x] Production application flow verified
- [x] WebSocket path verified
- [x] Deployment process documented

---

# 46. Current Status

Phase 09 is complete.

The production deployment workflow is now:

```text
CODE
 |
 v
GIT
 |
 v
CI/CD
 |
 v
DOCKER IMAGE
 |
 v
REGISTRY
 |
 v
MANUAL RELEASE
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
PRODUCTION
```

The next major requirement is controlled rollback.

---

# 47. Remaining Improvements

The following are intentionally not considered complete yet:

```text
[ ] Rollback testing
[ ] Automated deployment health check
[ ] Automatic rollback on health-check failure
[ ] DEV environment
[ ] TEST environment
[ ] Proper automated CI tests
[ ] Security hardening
[ ] SSH hardening
[ ] File-type SSH CI variable
[ ] Remove unnecessary public port 3000
[ ] Production secret management
[ ] Monitoring
[ ] Error/log management
[ ] MongoDB backup/recovery
```

---

# 48. Next Phase

The next phase is:

```text
application/phase-10-rollback.md
```

Phase 10 will document the rollback strategy:

```text
Production
    |
    v
Current Image
    |
    | Problem detected
    v
Previous Known-Good Image
    |
    v
docker compose pull
    |
    v
docker compose up -d
    |
    v
Health Check
    |
    v
Production Restored
```

The focus will be on **safe, deterministic rollback using commit-based Docker image tags**, including the actual rollback command pattern, verification, failure scenarios, and how rollback differs from redeployment.