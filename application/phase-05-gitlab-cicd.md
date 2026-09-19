# Phase 05 — GitLab CI/CD

## 1. Goal

The goal of this phase is to automate the Docker image build and production deployment process using GitLab CI/CD.

Before CI/CD, deployment was mostly manual:

```text
Developer
   |
   | git push
   v
GitLab
   |
   | manually build Docker image
   | manually push image
   | manually SSH into EC2
   | manually pull image
   | manually restart container
   v
Production
```

After CI/CD:

```text
Developer
   |
   | git push
   v
GitLab
   |
   +--> Build Docker image
   |
   +--> Push image to Container Registry
   |
   +--> Manual production deployment
   |
   +--> SSH to EC2
   |
   +--> Pull exact image
   |
   +--> Restart application
   v
Production
```

The important principle is:

> Build automatically, deploy production intentionally.

Production deployment was kept manual rather than deploying every push directly to production.

---

# 2. Why CI/CD?

Without CI/CD, every deployment requires repetitive manual work.

For example:

```bash
docker build
docker tag
docker push
ssh EC2
docker pull
docker compose up
```

This creates several problems:

- Manual mistakes
- Different deployment commands
- Difficult rollback
- No consistent deployment process
- More time spent on deployment
- Difficult deployment history
- Higher chance of human error

CI/CD moves this process into a repeatable pipeline.

---

# 3. Pipeline Design

The pipeline was divided into two stages:

```text
BUILD
  |
  v
DEPLOY
```

The build stage:

```text
Source Code
    |
    v
Docker Build
    |
    v
Docker Image
    |
    v
GitLab Container Registry
```

The deployment stage:

```text
GitLab CI
    |
    | SSH
    v
EC2
    |
    v
Docker Compose
    |
    v
Exact Image Version
    |
    v
Running Container
```

---

# 4. Important Deployment Principle

The production deployment should not depend on the `latest` Docker tag.

Instead, every commit should produce a unique image.

For example:

```text
commit A
    |
    +--> image:commit-A

commit B
    |
    +--> image:commit-B

commit C
    |
    +--> image:commit-C
```

GitLab provides:

```text
CI_COMMIT_SHA
```

This contains the commit SHA associated with the pipeline.

Therefore the Docker image can be tagged:

```text
$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

This creates a direct relationship between:

```text
Git Commit
    |
    v
Docker Image
    |
    v
Production Container
```

---

# 5. Why Use Commit SHA Tags?

Consider three deployments:

```text
Version A
Version B
Version C
```

If all of them use:

```text
latest
```

it becomes difficult to know exactly which image is running.

With commit-based tags:

```text
image:commit-A
image:commit-B
image:commit-C
```

we can identify the exact image associated with a Git commit.

This is important for:

- Traceability
- Reproducibility
- Deployment history
- Debugging
- Rollback

For example:

```text
Current:

image:commit-C

Rollback:

image:commit-B
```

Rollback will be covered in detail in Phase 10.

---

# 6. Initial GitLab CI Pipeline

The `.gitlab-ci.yml` file is stored in the root of the application repository.

The basic pipeline structure is:

```yaml
stages:
  - build
  - deploy

variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_COMMIT_SHA
```

This gives us:

```text
build
  |
  v
deploy
```

---

# 7. Docker Build Stage

Docker-in-Docker was used for building the Docker image inside GitLab CI.

Example:

```yaml
build:
  stage: build

  image: docker:29

  services:
    - name: docker:29-dind
      alias: docker

  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""

  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login "$CI_REGISTRY" -u "$CI_REGISTRY_USER" --password-stdin
    - docker build -t "$IMAGE_NAME:$IMAGE_TAG" .
    - docker push "$IMAGE_NAME:$IMAGE_TAG"
```

---

# 8. Understanding Docker-in-Docker

GitLab CI jobs normally run inside containers.

The build job needs access to Docker.

Therefore a Docker daemon is provided through:

```yaml
services:
  - name: docker:29-dind
    alias: docker
```

`dind` means:

```text
Docker-in-Docker
```

Conceptually:

```text
GitLab Runner
      |
      v
CI Job Container
      |
      | Docker CLI
      v
Docker-in-Docker Service
      |
      v
Docker Image
```

The following variables tell the Docker CLI where the Docker daemon is:

```yaml
DOCKER_HOST: tcp://docker:2375
DOCKER_TLS_CERTDIR: ""
```

---

# 9. GitLab Container Registry Authentication

GitLab provides built-in CI variables for registry authentication:

```text
CI_REGISTRY
CI_REGISTRY_IMAGE
CI_REGISTRY_USER
CI_REGISTRY_PASSWORD
```

The pipeline authenticates with:

```bash
echo "$CI_REGISTRY_PASSWORD" |
docker login "$CI_REGISTRY" \
  -u "$CI_REGISTRY_USER" \
  --password-stdin
```

The important principle is:

> Credentials should come from CI/CD variables, not from source code.

Never hardcode credentials:

```yaml
password: my-real-password
```

in the repository.

---

# 10. Building the Docker Image

The pipeline builds the image using:

```bash
docker build -t "$IMAGE_NAME:$IMAGE_TAG" .
```

Where:

```text
IMAGE_NAME = $CI_REGISTRY_IMAGE
IMAGE_TAG  = $CI_COMMIT_SHA
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

Example conceptually:

```text
registry.example.com/project/backend:abc123
```

The actual registry/project details are intentionally not documented here.

---

# 11. Pushing the Image

After the image is built:

```bash
docker push "$IMAGE_NAME:$IMAGE_TAG"
```

The image is pushed to the GitLab Container Registry.

The complete build flow is:

```text
Git Repository
      |
      v
GitLab CI
      |
      v
Docker Build
      |
      v
Commit SHA Tag
      |
      v
GitLab Container Registry
```

---

# 12. Production Deployment Strategy

Production deployment was intentionally configured as:

```text
MANUAL
```

The reason is:

```text
git push != production release
```

A developer may push code to `main` without wanting an immediate production deployment.

Therefore:

```text
git push
   |
   v
Pipeline starts
   |
   v
Docker image built
   |
   v
Image pushed to registry
   |
   v
Production deployment waits
   |
   v
Developer manually starts deployment
```

This creates a basic release-control boundary.

---

# 13. Production Deployment Job

The deployment job uses Alpine Linux and installs the SSH client.

Example:

```yaml
deploy_production:
  stage: deploy

  image: alpine:3.20

  before_script:
    - apk add --no-cache openssh-client
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - printf '%s\n' "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H "$EC2_HOST" >> ~/.ssh/known_hosts

  script:
    - |
      ssh "$EC2_USER@$EC2_HOST" "
        cd /home/ubuntu/support-assistant/backend &&
        IMAGE_TAG=$CI_COMMIT_SHA docker compose pull &&
        IMAGE_TAG=$CI_COMMIT_SHA docker compose up -d &&
        IMAGE_TAG=$CI_COMMIT_SHA docker compose ps
      "

  environment:
    name: production

  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never
```

The real production infrastructure values are intentionally replaced with placeholders in this public documentation.

---

# 14. GitLab CI/CD Variables

The deployment requires infrastructure-specific values.

These should be stored in GitLab CI/CD Variables.

Example:

```text
EC2_HOST
EC2_USER
SSH_PRIVATE_KEY
```

For public documentation, use:

```text
YOUR_EC2_IP
YOUR_EC2_USER
YOUR_SSH_PRIVATE_KEY
```

Never put the real values into Git.

---

# 15. SSH Deployment Flow

The deployment flow is:

```text
GitLab Runner
      |
      | SSH
      v
EC2 Server
      |
      v
Application Directory
      |
      v
Docker Compose
      |
      v
GitLab Container Registry
      |
      v
Docker Image
```

The CI runner connects to EC2 using SSH:

```bash
ssh "$EC2_USER@$EC2_HOST"
```

---

# 16. known_hosts

The deployment job runs:

```bash
ssh-keyscan -H "$EC2_HOST" >> ~/.ssh/known_hosts
```

This prepares the SSH `known_hosts` file.

The purpose is to avoid an interactive prompt such as:

```text
Are you sure you want to continue connecting?
```

This allows the deployment to run non-interactively.

For a hardened production implementation, host-key verification should be managed more carefully rather than blindly trusting a newly scanned key.

---

# 17. EC2 Deployment Commands

The deployment job executes commands on EC2.

Conceptually:

```bash
cd /home/ubuntu/support-assistant/backend

IMAGE_TAG=$CI_COMMIT_SHA docker compose pull

IMAGE_TAG=$CI_COMMIT_SHA docker compose up -d

IMAGE_TAG=$CI_COMMIT_SHA docker compose ps
```

The important part is:

```text
IMAGE_TAG=$CI_COMMIT_SHA
```

The same image version is used throughout the deployment.

---

# 18. Why IMAGE_TAG Is Required

The production Docker Compose file was changed from:

```yaml
image: registry.example.com/project/backend:latest
```

to:

```yaml
image: registry.example.com/project/backend:${IMAGE_TAG:?IMAGE_TAG is required}
```

This is intentional.

It prevents accidental deployment without explicitly specifying an image version.

If someone runs:

```bash
docker compose up -d
```

without:

```text
IMAGE_TAG
```

Compose should fail with an error similar to:

```text
required variable IMAGE_TAG is missing a value
```

This is a safety mechanism.

---

# 19. Build Once, Deploy the Same Image

The deployment architecture follows:

```text
SOURCE
  |
  v
BUILD
  |
  v
DOCKER IMAGE
  |
  +------> DEV
  |
  +------> TEST
  |
  +------> PROD
```

The important principle is:

> Build the artifact once and promote the same artifact between environments.

Do not rebuild the application separately for production.

Otherwise:

```text
DEV image != PROD image
```

and differences can appear between environments.

The future DEV / TEST / PROD strategy will be documented separately.

---

# 20. Problem Encountered: Repository Root

One of the initial pipeline attempts used:

```bash
cd server
```

inside the CI process.

The pipeline failed because the Git repository itself was already the backend/server directory.

The repository structure was conceptually:

```text
support-assistant-server/
    package.json
    Dockerfile
    src/
    tests/
```

Therefore the Docker build context was already:

```text
.
```

not:

```text
./server
```

The incorrect assumption caused the pipeline to look for a directory that did not exist.

### Lesson

Always verify the repository root before writing CI paths.

Useful local commands:

```powershell
git rev-parse --show-toplevel
```

and:

```powershell
Get-ChildItem
```

Inside CI, useful commands include:

```bash
pwd
ls
```

---

# 21. Test Stage Attempt

A test stage was also attempted.

The intended pipeline was:

```text
build
  |
  v
test
  |
  v
deploy
```

However, the existing integration and E2E tests were not yet designed as isolated CI tests.

Some tests expected:

```text
Running application server
```

and:

```text
Database
```

to already be available.

The CI environment did not yet provide those dependencies.

Therefore the test stage failed.

This did not mean testing was removed permanently.

Instead, automated testing was intentionally deferred until a proper CI test environment can be created.

The future target is:

```text
CI
 |
 +--> Unit Tests
 |
 +--> Integration Tests
 |
 +--> E2E Tests
 |
 v
Build
 |
 v
Deploy
```

### Important Lesson

A test command that works on a developer machine does not automatically become a valid CI test.

CI needs a controlled test environment.

For example:

```text
Integration Test
      |
      +--> Application
      |
      +--> Test Database
      |
      +--> Test Configuration
```

Testing will be implemented properly in a later phase.

---

# 22. Problem Encountered: docker compose ps

The deployment initially used:

```bash
IMAGE_TAG=$CI_COMMIT_SHA docker compose pull

IMAGE_TAG=$CI_COMMIT_SHA docker compose up -d

docker compose ps
```

The first two commands worked.

However, the final command did not receive:

```text
IMAGE_TAG
```

Because the Compose file intentionally requires:

```text
IMAGE_TAG
```

the final command failed.

The fix was:

```bash
IMAGE_TAG=$CI_COMMIT_SHA docker compose ps
```

The final deployment became:

```bash
IMAGE_TAG=$CI_COMMIT_SHA docker compose pull

IMAGE_TAG=$CI_COMMIT_SHA docker compose up -d

IMAGE_TAG=$CI_COMMIT_SHA docker compose ps
```

### Lesson

When a Compose configuration requires an environment variable, every Compose command that evaluates that configuration must receive the required variable.

---

# 23. Final Working Pipeline

The final pipeline follows:

```text
Developer
    |
    | git push
    v
GitLab
    |
    v
BUILD
    |
    +--> Docker Build
    |
    +--> Tag using CI_COMMIT_SHA
    |
    +--> Push to Registry
    |
    v
DEPLOY PRODUCTION
    |
    | Manual
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
docker compose ps
    |
    v
Production
```

---

# 24. Production Verification

After deployment, the production health endpoint was checked through HTTPS.

Example:

```bash
curl https://YOUR_DOMAIN/health
```

Expected response:

```json
{
  "status": "ok",
  "uptime": 123.45,
  "timestamp": "..."
}
```

The actual production domain is intentionally not stored in this public repository.

The verification flow is:

```text
GitLab
   |
   v
Image Built
   |
   v
Image Pushed
   |
   v
EC2 Pulled Image
   |
   v
Container Running
   |
   v
Nginx
   |
   v
HTTPS
   |
   v
/health = OK
```

---

# 25. What CI/CD Solved

Before CI/CD:

```text
Manual Build
Manual Tag
Manual Push
Manual SSH
Manual Pull
Manual Restart
Manual Verification
```

After CI/CD:

```text
Git Push
   |
   v
Automatic Build
   |
   v
Automatic Registry Push
   |
   v
Manual Production Approval
   |
   v
Automatic EC2 Deployment
```

This reduces repetitive deployment work and makes the process more consistent.

---

# 26. Security Rules

## Never commit secrets

Do not put the following into Git:

```text
.env
API keys
MongoDB passwords
JWT secrets
Encryption keys
SSH private keys
Registry passwords
Cloud credentials
```

Use:

```text
GitLab CI/CD Variables
```

for CI secrets.

---

# 27. Never Publish Production Configuration

Commands such as:

```bash
docker compose config
```

can render environment variables into the resulting configuration.

Therefore, do not paste complete production configuration output into:

- GitHub
- GitLab
- Public documentation
- README files
- Screenshots
- Public chat

A rendered production configuration can contain:

```text
Database credentials
API keys
JWT secrets
Encryption secrets
```

The `.env` file may be ignored correctly while a rendered configuration can still expose the same secrets.

---

# 28. SSH Key Handling

The initial implementation used a GitLab CI variable containing the SSH private key.

Conceptually:

```text
SSH_PRIVATE_KEY
```

The deployment job writes it to:

```text
~/.ssh/id_rsa
```

and applies restrictive permissions:

```bash
chmod 600 ~/.ssh/id_rsa
```

A future security improvement is to use a GitLab **file-type CI/CD variable** for the SSH private key.

This allows the key to be handled as a file instead of reconstructing it manually inside the job.

This improvement is intentionally tracked for the security phase.

---

# 29. Registry Authentication

The pipeline uses GitLab's built-in variables:

```text
CI_REGISTRY_USER
CI_REGISTRY_PASSWORD
```

instead of putting registry credentials into source code.

The principle is:

```text
Repository Credentials
        !=
Container Registry Credentials
```

Different credentials should have different purposes and scopes.

---

# 30. Important Decisions

## Decision 1 — Commit SHA image tags

Chosen:

```text
$CI_COMMIT_SHA
```

instead of relying on:

```text
latest
```

Reason:

- Traceability
- Reproducibility
- Easier rollback
- Clear relationship between source and artifact

---

## Decision 2 — Production deployment is manual

Chosen:

```yaml
when: manual
```

Reason:

```text
push != production release
```

The image is built automatically, while production deployment remains an intentional action.

---

## Decision 3 — Build image in CI

Chosen:

```text
GitLab CI
    |
    v
Docker Build
    |
    v
Container Registry
```

instead of building the image manually on EC2.

Reason:

- Consistent build environment
- Reproducibility
- Centralized artifact
- EC2 only needs to run the image

---

## Decision 4 — EC2 pulls the image

EC2 does not build the application source code.

It pulls the already-built image:

```text
Registry
    |
    v
EC2
```

This keeps production simpler.

---

## Decision 5 — Tests are deferred, not removed

The existing tests need a proper CI environment.

The future implementation will provide:

```text
Test Configuration
Test Database
Application Process
Unit Tests
Integration Tests
E2E Tests
```

before making tests a required deployment gate.

---

# 31. Lessons Learned

### Lesson 1

Always verify the repository root before writing CI paths.

### Lesson 2

`latest` is convenient but insufficient for controlled production deployment.

### Lesson 3

A Docker image should be treated as a deployment artifact.

### Lesson 4

Build once and deploy the same artifact.

### Lesson 5

Manual production approval provides a release boundary.

### Lesson 6

CI tests require proper infrastructure.

### Lesson 7

Required Compose variables must be supplied to every Compose command that evaluates the configuration.

### Lesson 8

Never expose rendered production configuration.

### Lesson 9

Infrastructure credentials and application credentials should be handled separately.

---

# 32. Verification Checklist

Before considering this phase complete:

- [x] GitLab CI file created
- [x] Build stage working
- [x] Docker-in-Docker configured
- [x] GitLab Container Registry authentication working
- [x] Docker image built in CI
- [x] Image tagged using commit SHA
- [x] Image pushed to registry
- [x] Production deployment configured
- [x] Deployment restricted to `main`
- [x] Production deployment configured as manual
- [x] SSH deployment working
- [x] EC2 pulls exact image
- [x] Docker Compose starts exact image
- [x] Compose requires `IMAGE_TAG`
- [x] Deployment verification performed
- [x] Production health endpoint verified
- [x] Repository-root path issue resolved
- [x] Test-stage limitation documented

---

# 33. Current Status

Phase 05 is complete.

The production deployment pipeline now follows:

```text
Developer
   |
   | git push
   v
GitLab
   |
   v
Build Docker Image
   |
   v
Tag = CI_COMMIT_SHA
   |
   v
GitLab Container Registry
   |
   v
Manual Production Deploy
   |
   v
SSH
   |
   v
EC2
   |
   v
Docker Compose
   |
   v
Exact Image
   |
   v
Production
```

---

# 34. Remaining Improvements

The following are intentionally not considered complete yet:

```text
[ ] Automated CI tests
[ ] Deployment health check inside CI
[ ] Automatic rollback on failed deployment
[ ] DEV environment
[ ] TEST environment
[ ] Production security hardening
[ ] SSH hardening
[ ] File-type SSH CI variable
[ ] Remove unnecessary public port 3000
[ ] Production secret management
[ ] Monitoring
[ ] Error management
[ ] MongoDB backup/recovery
```

These remain part of the overall project roadmap.

---

# 35. Next Phase

The next phase documents the AWS EC2 infrastructure setup.

Next file:

```text
application/phase-06-aws-ec2.md
```

Phase 06 will cover:

```text
AWS EC2
   |
   +--> Instance
   +--> Ubuntu
   +--> Elastic IP
   +--> SSH
   +--> Docker
   +--> Docker Compose
   +--> Security Group
   +--> Production application directory
   +--> MongoDB Atlas connectivity
```

The goal is to explain:

- What was configured
- Why EC2 was used
- What runs on EC2
- What does not run on EC2
- How EC2 connects to the Docker Registry
- How EC2 connects to MongoDB Atlas
- How CI/CD deploys to EC2
- What infrastructure decisions were made
- What remains to be improved