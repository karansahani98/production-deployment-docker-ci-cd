# Phase 04 — Container Registry

## Status

✅ Completed

## Purpose

In the previous phase, the Docker image was successfully built and tested locally.

The next problem was:

> How does the production server get the exact Docker image that was built?

Instead of building the application again on AWS, we introduced a container registry.

The flow becomes:

```text
Developer Machine
       |
       v
Docker Build
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
```

The registry acts as the central storage location for Docker images.

---

# 1. Why Use a Container Registry?

Without a registry, deployment could look like:

```text
Developer
    |
    v
Copy application to EC2
    |
    v
Build Docker image on EC2
    |
    v
Run application
```

This creates unnecessary work on the production server.

With a registry:

```text
Developer
    |
    v
Build image
    |
    v
Push image
    |
    v
Container Registry
    |
    v
EC2 pulls image
```

The production server does not need to build the application.

---

# 2. Registry Selected

The project uses the GitLab Container Registry.

The important distinction is:

```text
GitLab Repository
```

stores source code.

While:

```text
GitLab Container Registry
```

stores Docker images.

Conceptually:

```text
GitLab
│
├── Repository
│      └── Source Code
│
└── Container Registry
       └── Docker Images
```

---

# 3. Repository and Registry Are Separate Resources

Source code:

```text
gitlab.com/OWNER/REPOSITORY
```

Docker registry:

```text
registry.gitlab.com/OWNER/IMAGE
```

Use placeholders in public documentation.

Never publish the real private repository name or production configuration unless intentionally made public.

---

# 4. Create the Docker Image

The image was already created in Phase 03.

Generic command:

```powershell
docker build -t my-app .
```

Verify:

```powershell
docker images
```

Example:

```text
REPOSITORY   TAG      IMAGE ID
my-app       latest   <IMAGE_ID>
```

The actual image ID is environment-specific.

---

# 5. Docker Registry Login

Authenticate Docker with the registry:

```powershell
docker login registry.example.com
```

For GitLab specifically, the registry hostname follows this pattern:

```text
registry.gitlab.com
```

The actual credentials must never be written into this documentation.

---

# 6. Authentication Principle

The registry credential needs permission to perform the required Docker registry operations.

For the implementation, registry access was intentionally separated from Git repository access.

Conceptually:

```text
Credential A
    |
    v
Git Repository
Read / Write
```

and:

```text
Credential B
    |
    v
Container Registry
Read / Write
```

This follows the principle of least privilege.

A credential should not receive access that it does not need.

---

# 7. Verify Docker Login

After login:

```powershell
docker login registry.example.com
```

Expected:

```text
Login Succeeded
```

Docker stores registry authentication locally.

Do not publish:

```text
docker config
```

or any file containing registry credentials.

---

# 8. Tag the Image

A Docker image needs a registry-compatible name before it can be pushed.

Generic pattern:

```powershell
docker tag my-app:latest registry.example.com/OWNER/my-app:latest
```

For GitLab:

```powershell
docker tag my-app:latest registry.gitlab.com/OWNER/my-app:latest
```

The values:

```text
OWNER
my-app
```

are placeholders.

---

# 9. Understand Docker Image Naming

The general structure is:

```text
REGISTRY / NAMESPACE / IMAGE : TAG
```

Example:

```text
registry.example.com/company/backend:v1
```

Breakdown:

```text
registry.example.com
        |
        +-- Registry

company
        |
        +-- Namespace / Owner

backend
        |
        +-- Image name

v1
        |
        +-- Tag
```

---

# 10. Push the Image

After tagging:

```powershell
docker push registry.example.com/OWNER/my-app:latest
```

For GitLab:

```powershell
docker push registry.gitlab.com/OWNER/my-app:latest
```

The registry now contains the Docker image.

---

# 11. Verify Local Tags

Run:

```powershell
docker images
```

You should see both forms:

```text
my-app
registry.example.com/OWNER/my-app
```

They can refer to the same underlying image.

---

# 12. Why `latest` Is Not Enough

Initially, an image can be pushed as:

```text
:latest
```

For example:

```text
registry.example.com/OWNER/my-app:latest
```

However, `latest` does not tell us exactly which Git commit produced the image.

Suppose:

```text
Commit A → latest
```

Then later:

```text
Commit B → latest
```

Now:

```text
latest
```

points to Commit B.

The previous version is no longer obvious from the tag.

For production deployment, we need a stronger versioning strategy.

---

# 13. Commit SHA Image Tagging

The project therefore uses the Git commit SHA as the image version.

Generic example:

```text
registry.example.com/OWNER/my-app:<COMMIT_SHA>
```

For example:

```text
registry.example.com/OWNER/my-app:a1b2c3d4
```

The SHA above is only an example.

Do not copy real production commit identifiers into generic documentation unless needed.

---

# 14. Why Commit SHA?

The relationship becomes:

```text
Git Commit
     |
     v
Commit SHA
     |
     v
Docker Image Tag
```

Example:

```text
Git Commit
   abc123
      |
      v
Docker Image
   :abc123
```

Now we can identify exactly which source revision produced the image.

---

# 15. Get Current Git Commit SHA

From the application repository:

```powershell
git rev-parse HEAD
```

Short version:

```powershell
git rev-parse --short HEAD
```

Example:

```text
abc1234
```

The actual SHA is repository-specific.

---

# 16. Tag Image With Commit SHA

Generic:

```powershell
docker tag my-app:latest registry.example.com/OWNER/my-app:<COMMIT_SHA>
```

Example structure:

```powershell
docker tag my-app:latest registry.example.com/OWNER/my-app:abc1234
```

The SHA shown above is only an example.

---

# 17. Push Commit-SHA Image

```powershell
docker push registry.example.com/OWNER/my-app:<COMMIT_SHA>
```

Example:

```powershell
docker push registry.example.com/OWNER/my-app:abc1234
```

After this, the registry contains a versioned image.

---

# 18. Build and Tag in One Workflow

The local workflow can conceptually be:

```powershell
docker build -t my-app .
```

Then:

```powershell
docker tag my-app:latest registry.example.com/OWNER/my-app:<COMMIT_SHA>
```

Then:

```powershell
docker push registry.example.com/OWNER/my-app:<COMMIT_SHA>
```

Flow:

```text
Source Code
    |
    v
Docker Build
    |
    v
Local Image
    |
    v
Commit SHA Tag
    |
    v
Registry
```

---

# 19. `latest` vs Commit SHA

## `latest`

```text
registry.example.com/OWNER/my-app:latest
```

Advantages:

- Easy to understand
- Convenient for simple development deployments

Problem:

```text
latest
```

changes over time.

---

## Commit SHA

```text
registry.example.com/OWNER/my-app:<COMMIT_SHA>
```

Advantages:

- Immutable reference
- Traceable to source
- Easier rollback
- Clear deployment history
- Better CI/CD behavior

This is the strategy selected for the production deployment pipeline.

---

# 20. Build Once, Deploy Later

The desired architecture is:

```text
Git Commit
     |
     v
CI Docker Build
     |
     v
Versioned Image
     |
     v
Container Registry
     |
     +----------+
     |          |
     v          v
    DEV       TEST
                |
                v
               PROD
```

The image is built once.

The environment decides where the image runs.

---

# 21. Why EC2 Should Pull the Image

The production server should not need:

```text
Source code
```

to build the application.

Instead:

```text
EC2
 |
 | docker pull
 v
Container Registry
 |
 v
Docker Image
 |
 v
Container
```

This separates:

```text
Build responsibility
```

from:

```text
Runtime responsibility
```

---

# 22. EC2 Registry Pull

The production server later uses:

```bash
docker pull registry.example.com/OWNER/my-app:<COMMIT_SHA>
```

This command belongs to the production deployment phase.

The important idea is:

```text
EC2 does not build the image.
EC2 pulls the already-built image.
```

---

# 23. Registry Authentication on EC2

If the registry is private, EC2 needs permission to pull the image.

Generic command:

```bash
docker login registry.example.com
```

The credentials must be provided securely.

Never store them in:

```text
docker-compose.yml
Git repository
README.md
shell history
public documentation
```

where possible.

---

# 24. CI Registry Authentication

Later, GitLab CI/CD can use GitLab's built-in registry variables:

```text
CI_REGISTRY
CI_REGISTRY_USER
CI_REGISTRY_PASSWORD
```

The pipeline can authenticate with:

```bash
echo "$CI_REGISTRY_PASSWORD" | docker login "$CI_REGISTRY" -u "$CI_REGISTRY_USER" --password-stdin
```

This avoids hardcoding registry credentials in the CI configuration.

---

# 25. Why `--password-stdin`?

Instead of:

```bash
docker login -u USER -p PASSWORD
```

use:

```bash
echo "$PASSWORD" | docker login REGISTRY -u USER --password-stdin
```

This reduces the chance of exposing the password through command-line arguments.

The CI/CD pipeline later uses this approach.

---

# 26. Verify Registry Image

After pushing, verify the image in the GitLab Container Registry UI.

Look for:

```text
Repository
   |
   v
Container Registry
   |
   v
Image
   |
   v
Tag
```

The important tag is the commit SHA.

---

# 27. Registry Image Digest

A registry can identify an image using a digest.

Conceptually:

```text
Image Tag
    |
    v
Image Manifest
    |
    v
SHA256 Digest
```

Example format:

```text
sha256:<DIGEST>
```

The exact digest is generated by the registry.

A digest provides a content-addressed reference to the image.

---

# 28. Tag vs Digest

A tag:

```text
my-app:<COMMIT_SHA>
```

is a human-friendly deployment reference.

A digest:

```text
my-app@sha256:<DIGEST>
```

is a content-addressed image reference.

The project primarily uses commit-SHA tags for deployment tracking.

---

# 29. Security Considerations

Never publish:

```text
docker login output containing secrets
registry passwords
registry access tokens
private registry credentials
Docker config files
```

Also avoid publishing:

```powershell
docker inspect ...
```

without reviewing the output.

Environment variables can appear in container metadata.

---

# 30. Credential Separation

The implementation intentionally separated:

```text
Git Repository Access
```

from:

```text
Docker Registry Access
```

This gives us:

```text
Git Credential
     |
     +-- repository operations

Registry Credential
     |
     +-- image push/pull
```

If one credential is compromised, its permissions are limited to its intended purpose.

---

# 31. Registry Workflow Completed

The implemented workflow was:

```text
Docker Image
     |
     v
Docker Tag
     |
     v
GitLab Container Registry
     |
     v
Registry Image
```

The image was successfully pushed to the registry.

This established the artifact source for the future EC2 deployment.

---

# 32. Problem / Learning During This Phase

The important operational distinction was:

```text
GitLab repository
```

versus:

```text
GitLab Container Registry
```

They are related through GitLab but serve different purposes.

The repository stores:

```text
Source Code
```

The registry stores:

```text
Docker Images
```

---

# 33. Registry Authentication Lesson

Authentication should be scoped to the operation being performed.

For example:

```text
Git repository
    |
    +-- source read/write

Container registry
    |
    +-- image read/write
```

This is preferable to using one highly privileged credential for everything.

---

# 34. Phase Verification

The following workflow was verified:

```text
[✓] Docker image exists locally
[✓] Registry authentication works
[✓] Image tagged for registry
[✓] Image pushed
[✓] Registry image available
[✓] Commit-SHA tagging understood
[✓] Registry authentication separated from Git authentication
[✓] Registry established as deployment artifact source
```

---

# 35. Final Architecture After Phase 04

The system now has:

```text
                 GitLab
                    |
          +---------+---------+
          |                   |
          v                   v
     Git Repository      Container Registry
          |                   |
          |                   |
          v                   v
      Source Code       Versioned Docker Image
                              |
                              |
                              v
                         Future AWS EC2
```

The next phase connects these pieces through CI/CD.

---

# 36. Phase Completion

```text
✅ PHASE 04 — CONTAINER REGISTRY COMPLETED
```

---

# 37. Next Phase

➡️ **Phase 05 — GitLab CI/CD**

The next phase will automate the process:

```text
git push
   |
   v
GitLab CI
   |
   v
Docker Build
   |
   v
Commit SHA Image
   |
   v
Container Registry
   |
   v
Manual Production Deployment
```

We will document the actual CI/CD implementation, including:

- `.gitlab-ci.yml`
- Docker-in-Docker
- CI variables
- Registry authentication
- Commit SHA tagging
- EC2 SSH deployment
- Manual production deployment
- CI path problem
- Compose `IMAGE_TAG` requirement
- Deployment verification
- Pipeline failures and fixes
- Security considerations

No live credentials or infrastructure secrets will be included.