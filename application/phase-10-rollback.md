# Phase 10 — Rollback

## 1. Goal

The goal of this phase is to document how a previous known-good Docker image can be restored when a production deployment introduces a problem.

The deployment architecture uses commit-based Docker image tags.

Therefore, each deployment can be identified by its Git commit.

```text
Git Commit
    |
    v
Docker Image
    |
    v
Container Registry
    |
    v
Production
```

If the current production version has a problem, the deployment can be changed back to a previous image.

The basic rollback flow is:

```text
Current Production
       |
       v
Problem Detected
       |
       v
Identify Previous Known-Good Image
       |
       v
Pull Previous Image
       |
       v
Start Previous Version
       |
       v
Health Check
       |
       v
Production Restored
```

---

# 2. Why Rollback Is Important

A deployment can succeed technically while still introducing an application problem.

For example:

```text
Docker Build       -> SUCCESS
Registry Push      -> SUCCESS
Container Start    -> SUCCESS
Application        -> PROBLEM
```

The container may be running while an important feature is broken.

Without rollback:

```text
Problem
   |
   v
Debug production
   |
   v
Create another fix
   |
   v
Build
   |
   v
Deploy
```

This can take significant time.

With a known-good image:

```text
Problem
   |
   v
Select previous image
   |
   v
Deploy previous image
   |
   v
Restore service
```

Rollback provides a faster recovery path.

---

# 3. Why Commit SHA Tags Make Rollback Easier

The deployment process does not rely on:

```text
latest
```

Instead it uses:

```text
CI_COMMIT_SHA
```

For example:

```text
image:commit-A
image:commit-B
image:commit-C
```

Suppose production currently runs:

```text
image:commit-C
```

and `commit-B` was the previous known-good release.

Rollback becomes:

```text
commit-C
   |
   | problem
   v
commit-B
   |
   v
production
```

The image itself does not need to be rebuilt.

---

# 4. Rollback vs Redeployment

These are different operations.

## Redeployment

Redeployment means:

```text
Deploy the intended new version
```

Example:

```text
commit-D
```

## Rollback

Rollback means:

```text
Return to a previous known-good version
```

Example:

```text
commit-C
```

The important distinction is:

```text
Redeployment
    |
    v
New version

Rollback
    |
    v
Previous version
```

---

# 5. Build Once, Roll Back by Image

The preferred rollback model is:

```text
Git Commit
    |
    v
Docker Build
    |
    v
Immutable Image
    |
    v
Registry
    |
    +------> Production
    |
    +------> Rollback
```

The previous image already exists in the registry.

Therefore rollback does not require rebuilding the source code.

---

# 6. Example Deployment History

Assume these commits were deployed:

```text
Commit A
Commit B
Commit C
```

The registry contains:

```text
IMAGE:A
IMAGE:B
IMAGE:C
```

Production currently runs:

```text
IMAGE:C
```

Then a problem is discovered.

The previous known-good image is:

```text
IMAGE:B
```

Rollback means changing:

```text
IMAGE:C
```

to:

```text
IMAGE:B
```

---

# 7. Identify the Current Version

Before rollback, determine which version is currently running.

On EC2:

```bash
docker compose ps
```

Then inspect the image:

```bash
docker ps
```

You can also inspect the container:

```bash
docker inspect YOUR_CONTAINER_NAME
```

The goal is to identify:

```text
Current image
Current image tag
Current container
```

Do not rely only on memory.

Verify the actual running image.

---

# 8. Identify the Previous Known-Good Version

A rollback should target a version that is known to have worked.

Possible sources include:

```text
GitLab pipeline history
Deployment history
Previous production deployment
Container image tags
Git commit history
```

The decision should be based on a verified previous release rather than simply choosing an older commit at random.

Conceptually:

```text
Current:
commit-C

Previous known-good:
commit-B
```

---

# 9. Check Registry Image

Before attempting rollback, verify that the previous image still exists in the Container Registry.

Conceptually:

```bash
docker pull YOUR_REGISTRY_IMAGE:YOUR_PREVIOUS_COMMIT_SHA
```

If the pull succeeds:

```text
Previous image exists
```

If it fails:

```text
Previous image unavailable
```

In that case, do not assume rollback is possible until the image availability problem is resolved.

---

# 10. Rollback Environment Variable

The production Compose file requires:

```text
IMAGE_TAG
```

Example:

```yaml
image: YOUR_REGISTRY_IMAGE:${IMAGE_TAG:?IMAGE_TAG is required}
```

Therefore rollback simply changes the value of:

```text
IMAGE_TAG
```

from:

```text
CURRENT_COMMIT_SHA
```

to:

```text
PREVIOUS_COMMIT_SHA
```

This is one of the main benefits of using explicit image tags.

---

# 11. Manual Rollback Command

On EC2, navigate to the production deployment directory:

```bash
cd /home/YOUR_EC2_USER/YOUR_APPLICATION/backend
```

Then pull the previous image:

```bash
IMAGE_TAG=YOUR_PREVIOUS_COMMIT_SHA docker compose pull
```

Then start the previous version:

```bash
IMAGE_TAG=YOUR_PREVIOUS_COMMIT_SHA docker compose up -d
```

Then verify:

```bash
IMAGE_TAG=YOUR_PREVIOUS_COMMIT_SHA docker compose ps
```

---

# 12. Complete Rollback Sequence

The basic rollback sequence is:

```bash
cd /home/YOUR_EC2_USER/YOUR_APPLICATION/backend

IMAGE_TAG=YOUR_PREVIOUS_COMMIT_SHA docker compose pull

IMAGE_TAG=YOUR_PREVIOUS_COMMIT_SHA docker compose up -d

IMAGE_TAG=YOUR_PREVIOUS_COMMIT_SHA docker compose ps
```

The actual production path and image name are intentionally replaced with placeholders.

---

# 13. Verify Container After Rollback

Check:

```bash
docker compose ps
```

The backend should be running.

Then inspect:

```bash
docker ps
```

Confirm that the container is using the expected image.

The target state is:

```text
Container
    |
    v
Previous Known-Good Image
```

---

# 14. Check Application Logs

After rollback:

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
Unexpected errors
```

The purpose is to confirm that the previous version started normally.

---

# 15. Local Health Check After Rollback

From EC2:

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
Previous Image
      |
      v
Docker
      |
      v
Node.js
      |
      v
Application
```

---

# 16. Public HTTPS Health Check

After the local health check succeeds:

```bash
curl https://api.example.com/health
```

Expected:

```json
{
  "status": "ok"
}
```

The actual production domain is intentionally replaced with:

```text
api.example.com
```

in this public documentation.

---

# 17. Functional Verification

A health endpoint alone is not enough for every rollback.

The application should also be checked for the functionality affected by the failed deployment.

For example:

```text
Authentication
    |
    v
API request
    |
    v
Database operation
    |
    v
WebSocket
    |
    v
External service
```

The exact verification depends on what caused the production issue.

---

# 18. Complete Rollback Verification

After rollback:

```text
1. Container running
       |
       v
2. Correct image tag
       |
       v
3. Application logs healthy
       |
       v
4. Local /health passes
       |
       v
5. HTTPS /health passes
       |
       v
6. Affected feature verified
       |
       v
7. Production restored
```

Do not consider rollback complete simply because the container started.

---

# 19. Example Rollback Scenario

Suppose:

```text
Production:
commit-C
```

A new deployment introduced a problem.

The deployment history is:

```text
commit-A -> successful
commit-B -> successful
commit-C -> successful
commit-D -> problematic
```

Production is currently:

```text
commit-D
```

The known-good version is:

```text
commit-C
```

Rollback:

```text
commit-D
   |
   | rollback
   v
commit-C
```

The image for `commit-C` already exists.

Therefore:

```text
No rebuild
No source checkout
No npm install
```

is required on EC2.

The server simply pulls and runs the previous image.

---

# 20. Why Not Rebuild the Previous Version?

Suppose the original image was:

```text
image:commit-C
```

A rollback should use that existing image.

Do not unnecessarily do:

```text
Git checkout commit-C
       |
       v
Docker build
       |
       v
New image
```

That creates a different artifact.

The preferred model is:

```text
Original Build
      |
      v
Original Image
      |
      +----> Production
      |
      +----> Rollback
```

This keeps the rollback artifact identical to the original artifact.

---

# 21. Immutable Artifact Principle

A production Docker image should be treated as immutable.

Conceptually:

```text
commit-C
   |
   v
image:commit-C
```

Once pushed, that image should not be replaced with different application content under the same tag.

This is why commit-based tags are useful.

The tag represents a specific build artifact.

---

# 22. Do Not Overwrite Version Tags

Avoid workflows such as:

```text
commit-C -> image:release
```

and later:

```text
commit-D -> image:release
```

because the same tag now refers to different application content over time.

Prefer:

```text
commit-C -> image:commit-C
commit-D -> image:commit-D
```

Each version remains independently identifiable.

---

# 23. Rollback Through GitLab CI/CD

The current deployment flow uses a manual production deployment.

A future rollback can also be implemented as a manual GitLab CI/CD job.

Conceptually:

```text
GitLab
   |
   +--> Build
   |
   +--> Deploy Production
   |
   +--> Rollback Production
```

The rollback job would receive a specific image tag and deploy it to EC2.

For example:

```text
ROLLBACK_IMAGE_TAG
```

could identify the previous known-good version.

This should be introduced only after manual rollback has been tested successfully.

---

# 24. Manual Rollback First

Manual rollback should be understood before automation.

The sequence is:

```text
Manual Rollback
       |
       v
Verify
       |
       v
Automate
```

This is safer than immediately creating an automatic rollback process that has not been tested.

---

# 25. Future Automated Rollback

Once health checks are integrated into CI/CD, the future deployment architecture can become:

```text
Deploy New Image
       |
       v
Health Check
       |
       +-------- PASS --------+
       |                       |
       v                       v
   Deployment              Success
    Complete
       |
       |
       +-------- FAIL --------+
                                |
                                v
                         Previous Image
                                |
                                v
                         Health Check
                                |
                                v
                           Recovery
```

This should only be implemented after the manual rollback process is proven.

---

# 26. Rollback Trigger Examples

A rollback may be considered when a deployment causes:

```text
Application startup failure
Critical API failure
Authentication failure
Database compatibility issue
WebSocket failure
Severe performance regression
External integration failure
Unexpected application errors
```

The exact rollback criteria should be defined by the application and operational requirements.

---

# 27. Database Rollback Warning

Application rollback and database rollback are not automatically the same thing.

For example:

```text
Application Version B
       |
       v
Database Schema B
```

Then:

```text
Application Version C
       |
       v
Database Schema C
```

If version C modifies the database schema, simply returning to application version B may not be safe.

Therefore:

```text
Application Rollback
        !=
Database Rollback
```

Database migrations must be designed with backward compatibility in mind.

---

# 28. Backward-Compatible Database Changes

A safer migration strategy is:

```text
Old Application
      |
      v
Compatible Database Change
      |
      v
New Application
```

For example, adding a nullable column is generally easier to roll back than immediately removing or renaming an existing column.

The exact migration strategy depends on the application.

The important lesson is:

> A Docker rollback does not automatically undo database changes.

---

# 29. External Services and Rollback

The application may depend on external APIs.

Therefore rollback should consider:

```text
Application version
+
Database compatibility
+
External API compatibility
```

A previous application version may still fail if an external dependency has changed incompatibly.

Rollback therefore requires application-level verification.

---

# 30. Preserve Previous Images

Do not immediately delete previous production images.

Keep enough historical versions available for recovery.

Conceptually:

```text
Registry
   |
   +--> image:A
   +--> image:B
   +--> image:C
   +--> image:D
```

The retention policy should balance:

```text
Rollback capability
+
Registry storage cost
```

A future cleanup policy can remove very old images after an appropriate retention period.

---

# 31. Deployment History

Maintain a clear deployment history:

```text
Date
Commit
Image
Environment
Deployment Result
```

For example:

```text
Deployment
   |
   +--> Commit SHA
   +--> Image Tag
   +--> Environment
   +--> Status
```

This helps answer:

```text
What is running?
What was running before?
Which version should we roll back to?
```

---

# 32. Rollback Checklist

Before rollback:

```text
[ ] Confirm production problem
[ ] Identify current image
[ ] Identify previous known-good image
[ ] Confirm previous image exists
[ ] Check database compatibility
[ ] Check external dependency compatibility
```

During rollback:

```text
[ ] Pull previous image
[ ] Start previous image
[ ] Check container status
[ ] Check logs
```

After rollback:

```text
[ ] Local health check
[ ] HTTPS health check
[ ] Verify affected functionality
[ ] Verify WebSocket if relevant
[ ] Confirm production recovery
[ ] Record rollback
```

---

# 33. Rollback Troubleshooting

## Problem 1 — Previous image cannot be pulled

Check:

```text
Registry authentication
Image name
Image tag
Registry retention
Network connectivity
```

---

## Problem 2 — Previous container does not start

Check:

```bash
docker compose logs backend
```

Potential causes:

```text
Environment configuration
Database compatibility
Missing dependency
Port conflict
Application startup error
```

---

## Problem 3 — Container runs but application is unhealthy

Check:

```bash
curl http://127.0.0.1:3000/health
```

Then:

```bash
docker compose logs backend
```

The image may be running while a dependency is unavailable.

---

## Problem 4 — Local health works but HTTPS fails

Check:

```text
Nginx
DNS
TLS
Security Group
```

The problem may not be related to the application image.

---

# 34. Important Decisions

## Decision 1 — Use commit SHA for rollback

Rollback targets:

```text
Previous Commit SHA
```

rather than:

```text
latest
```

---

## Decision 2 — Roll back the existing image

The previous image is pulled from the registry.

It is not rebuilt.

---

## Decision 3 — Manual rollback before automation

The rollback process should first be proven manually.

---

## Decision 4 — Verify after rollback

Rollback is not complete until the application is verified.

---

## Decision 5 — Consider database compatibility

Application rollback must consider database schema changes.

---

# 35. Lessons Learned

### Lesson 1

Rollback is much easier when deployments use immutable image tags.

### Lesson 2

A Docker image is a deployment artifact and should be preserved.

### Lesson 3

Rollback should not require rebuilding the application.

### Lesson 4

A running container does not mean the application is healthy.

### Lesson 5

Database changes can make application rollback more complicated.

### Lesson 6

Health checks are necessary for reliable rollback decisions.

### Lesson 7

Manual rollback should be understood before automatic rollback is introduced.

### Lesson 8

Previous known-good versions should remain available for a reasonable retention period.

### Lesson 9

Production recovery should be measurable and verifiable.

---

# 36. Verification Checklist

Before considering this phase complete:

- [x] Commit-based image tags documented
- [x] Current image identification documented
- [x] Previous image identification documented
- [x] Previous image pull documented
- [x] Rollback Compose commands documented
- [x] Container verification documented
- [x] Application health verification documented
- [x] HTTPS verification documented
- [x] Functional verification documented
- [x] Database rollback warning documented
- [x] External dependency considerations documented
- [x] Future automated rollback documented

---

# 37. Current Status

Phase 10 is complete.

The rollback strategy is:

```text
Current Production
       |
       v
Identify Problem
       |
       v
Identify Previous Known-Good Commit
       |
       v
Pull Previous Docker Image
       |
       v
Start Previous Image
       |
       v
Check Container
       |
       v
Check Application
       |
       v
Check HTTPS
       |
       v
Verify Critical Functionality
       |
       v
Production Restored
```

The most important property is:

```text
Rollback = Run an existing known-good image
```

not:

```text
Rollback = Rebuild old source code
```

---

# 38. Remaining Improvements

The following are intentionally not considered complete yet:

```text
[ ] Perform a real rollback test
[ ] Add deployment health check to CI/CD
[ ] Add automatic rollback after failed health check
[ ] Define image retention policy
[ ] Define database migration rollback strategy
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

# 39. Next Phase

The next phase is:

```text
application/phase-11-security.md
```

Phase 11 will document production security, including:

```text
Secrets
   |
   +--> Environment variables
   +--> CI/CD variables
   +--> SSH keys
   +--> Registry credentials
   +--> Database credentials
   +--> API keys

Network
   |
   +--> Security Groups
   +--> Public ports
   +--> SSH access
   +--> HTTPS

Server
   |
   +--> Docker permissions
   +--> OS updates
   +--> SSH hardening

Application
   |
   +--> Secrets
   +--> Authentication
   +--> CORS
   +--> WebSocket security

CI/CD
   |
   +--> Protected variables
   +--> Credential scopes
   +--> Deployment permissions
```

The security phase will also document the production-secret exposure incident as a **sanitized engineering lesson**, without publishing any actual credentials or sensitive values.