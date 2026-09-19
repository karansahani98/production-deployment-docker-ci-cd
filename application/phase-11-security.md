# Phase 11 — Security

## Goal

Make the deployment secure enough for a real production environment.

This phase covers the security controls around:

- Application secrets
- GitLab credentials
- Container Registry credentials
- SSH access
- AWS Security Groups
- Nginx
- HTTPS
- WebSocket connections
- JWT authentication
- CORS
- MongoDB Atlas
- Docker containers
- Dependency management
- Logging
- Least privilege
- Credential exposure prevention

The objective is not to make a system "100% secure".

The objective is to reduce unnecessary attack surface and establish a repeatable security process.

---

# 1. Security Model

The production request flow is:

    Internet
       |
       v
    HTTPS
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


There are several security boundaries:

1. Internet → Nginx
2. Nginx → Docker
3. Application → MongoDB Atlas
4. GitLab CI/CD → Container Registry
5. GitLab CI/CD → EC2
6. Users → Application Authentication
7. Application → External APIs

Each boundary needs its own controls.

---

# 2. Never Commit Secrets

The most important rule:

> Secrets must never be stored in Git.

Examples of secrets:

- MongoDB URI
- MongoDB password
- JWT secret
- Encryption secret
- API keys
- AssemblyAI API key
- Groq API key
- Anthropic API key
- External service credentials
- SSH private keys
- GitLab Personal Access Tokens

The repository should contain configuration templates only.

Example:

    MONGODB_URI=YOUR_MONGODB_URI
    JWT_SECRET=YOUR_JWT_SECRET
    API_KEY=YOUR_API_KEY

Never commit:

    .env

The production `.env` remains on the EC2 server and is not committed to Git.

---

# 3. .gitignore

The application repository should ignore sensitive and local files.

Example:

    .env
    .env.*
    node_modules/
    npm-debug.log
    *.pem
    *.key

The exact `.gitignore` should be reviewed according to the application requirements.

Verify:

    git status

Sensitive files should not appear as untracked files.

---

# 4. Docker Build Security

The Docker build must not copy `.env` into the image.

The `.dockerignore` contains:

    node_modules
    .env
    npm-debug.log
    .git
    .gitignore
    Dockerfile
    .dockerignore

The important rule is:

> Secrets should be injected at runtime, not baked into the Docker image.

For example:

    docker run --env-file .env image-name

or Docker Compose:

    env_file:
      - .env

This keeps the image reusable across environments.

---

# 5. Why Secrets Should Not Be Inside Docker Images

Consider this bad approach:

    ENV JWT_SECRET=some-secret

The secret becomes part of the image configuration.

Anyone who can inspect or obtain the image may potentially discover information that should not be public.

The preferred architecture is:

    Docker Image
          |
          | no environment-specific secrets
          |
          v
    Runtime Environment
          |
          +---- MONGODB_URI
          +---- JWT_SECRET
          +---- API_KEYS
          +---- ENCRYPTION_SECRET

This also allows the same image to run in:

    DEV
    TEST
    PROD

with different configuration.

---

# 6. Production .env

The production `.env` exists only on the EC2 server.

Example location:

    /home/ubuntu/support-assistant/backend/.env

Docker Compose reads it using:

    env_file:
      - .env

The `.env` file should have restrictive permissions.

Example:

    chmod 600 /home/ubuntu/support-assistant/backend/.env

Verify:

    ls -la /home/ubuntu/support-assistant/backend/.env

The objective is that only the required operating-system user can read the file.

Never paste production `.env` contents into:

- GitLab issues
- GitHub
- README files
- public documentation
- screenshots
- support tickets
- chat messages
- logs

---

# 7. CI/CD Secrets

GitLab CI/CD requires credentials to perform deployment.

Examples:

    SSH_PRIVATE_KEY
    EC2_HOST
    EC2_USER

These should be stored as GitLab CI/CD variables.

The actual values must not be hard-coded into:

    .gitlab-ci.yml

Bad:

    ssh ubuntu@13.x.x.x

Good:

    ssh "$EC2_USER@$EC2_HOST"

This allows the repository to remain safe if it becomes public.

---

# 8. GitLab Registry Authentication

GitLab provides predefined CI variables such as:

    CI_REGISTRY
    CI_REGISTRY_USER
    CI_REGISTRY_PASSWORD
    CI_REGISTRY_IMAGE

The pipeline can authenticate using:

    echo "$CI_REGISTRY_PASSWORD" |
      docker login "$CI_REGISTRY" \
      -u "$CI_REGISTRY_USER" \
      --password-stdin

This avoids putting registry credentials directly inside the repository.

---

# 9. Personal Access Token Separation

During the implementation, two different GitLab Personal Access Tokens were used for different purposes.

## Repository token

Used for Git operations.

Required scopes were limited to:

    read_repository
    write_repository

## Container Registry token

Used for Docker Registry operations.

Required scopes were limited to:

    read_registry
    write_registry

This follows the principle:

> One credential should have only the permissions required for its job.

Do not use a full-access token when a limited-scope token is sufficient.

---

# 10. Why Credential Separation Matters

Suppose a Registry token is compromised.

If that token only has:

    read_registry
    write_registry

the potential impact is much smaller than a token that can:

- modify repository code
- delete projects
- manage users
- access unrelated resources

This is called:

> Least privilege

Always prefer the smallest permission set that works.

---

# 11. SSH Private Key

GitLab CI/CD needs SSH access to EC2 for deployment.

The private key is stored as a CI/CD variable:

    SSH_PRIVATE_KEY

It must never be committed to Git.

Never create:

    deploy-key.pem

inside the repository.

Never commit:

    *.pem

to GitHub or GitLab.

---

# 12. SSH Key Handling Improvement

The initial CI/CD implementation used a multiline CI variable and wrote it to:

    ~/.ssh/id_rsa

Example pattern:

    printf '%s\n' "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa

This worked for the deployment.

A future improvement is to use a GitLab **File-type CI/CD variable** for the private key.

Then the pipeline can directly reference the temporary file supplied by GitLab.

This reduces manual handling of multiline private-key content.

---

# 13. SSH Host Verification

The CI job uses:

    ssh-keyscan -H "$EC2_HOST" >> ~/.ssh/known_hosts

This allows SSH to recognize the EC2 host.

The purpose is to avoid interactive host-key confirmation during CI/CD.

For a stronger security model, the expected host key can be managed explicitly instead of dynamically accepting whatever key is returned by `ssh-keyscan`.

This is an area for future hardening.

---

# 14. AWS Security Group

The EC2 Security Group should expose only the ports that are actually required.

Typical production ports:

    80      HTTP
    443     HTTPS
    22      SSH

Application port:

    3000

should generally not be publicly accessible when Nginx is the public entry point.

The desired traffic path is:

    Internet
       |
       +---- 80/443
               |
               v
             Nginx
               |
               v
          127.0.0.1:3000

Port 3000 does not need to be exposed directly to the Internet.

---

# 15. Remove Public Port 3000

The Docker Compose configuration currently maps:

    3000:3000

This is useful for initial deployment/testing.

However, Nginx is now the public entry point.

The AWS Security Group should not allow:

    TCP 3000
    Source: 0.0.0.0/0

If port 3000 is still allowed publicly, remove that inbound rule.

After removing it, verify:

    HTTPS -> Nginx -> Docker -> Node.js

still works.

---

# 16. Why Nginx Protects the Application Boundary

Without Nginx:

    Internet -> :3000 -> Node.js

With Nginx:

    Internet -> HTTPS -> Nginx -> Node.js

Nginx becomes the controlled public entry point.

It handles:

- TLS termination
- HTTP/HTTPS
- reverse proxying
- WebSocket forwarding
- domain routing
- request headers

The application itself can remain behind the reverse proxy.

---

# 17. HTTPS

Production API traffic uses HTTPS.

Example:

    https://api.example.com

instead of:

    http://api.example.com

TLS protects traffic between:

    Client <-> Nginx

This is especially important for:

- Login credentials
- JWT tokens
- User information
- API requests
- WebSocket authentication
- Application responses

---

# 18. WebSocket Security

The application uses WebSockets for streaming.

Nginx must forward the upgrade headers.

Example:

    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

The production client should use secure WebSockets:

    wss://api.example.com/stream

instead of:

    ws://api.example.com/stream

when the connection is made through HTTPS.

The goal is:

    HTTPS + WSS

for production communication.

---

# 19. JWT Security

The application uses JWT authentication.

The JWT secret must be stored as an environment variable.

Example:

    JWT_SECRET=YOUR_LONG_RANDOM_SECRET

Never:

    const JWT_SECRET = "my-secret";

inside source code.

The application should reject requests where the token is:

- missing
- invalid
- expired

JWT secrets should be sufficiently random and should not be reused across unrelated environments.

---

# 20. JWT Environment Separation

DEV should not necessarily use the same JWT secret as PROD.

Example:

    DEV
    JWT_SECRET=DEV_SECRET

    TEST
    JWT_SECRET=TEST_SECRET

    PROD
    JWT_SECRET=PROD_SECRET

If a development environment is compromised, this prevents the same secret from automatically affecting production authentication.

---

# 21. CORS

CORS should be configured deliberately.

Do not blindly allow every origin in production.

Avoid:

    origin: "*"

for authenticated applications unless there is a specific reason and the security implications are understood.

Production should allow only the origins that actually need access.

Example concept:

    https://your-frontend.example.com

The exact allowed origins depend on the actual deployment architecture.

---

# 22. Authentication vs CORS

CORS is not authentication.

CORS controls which browser origins are allowed to make browser-based cross-origin requests.

JWT authentication controls whether the request is authenticated.

They solve different problems.

Conceptually:

    CORS
      |
      +---- Is this browser origin allowed?

    JWT
      |
      +---- Is this user authenticated?

Both may be required.

---

# 23. MongoDB Atlas Network Access

MongoDB is hosted outside EC2 using MongoDB Atlas.

The application connects using:

    MONGODB_URI

Atlas network access should allow only the required sources.

During deployment, the EC2 public/Elastic IP was added to the Atlas allowlist.

The security goal is:

    EC2
      |
      | allowed network access
      v
    MongoDB Atlas

Avoid unnecessarily allowing:

    0.0.0.0/0

to MongoDB.

---

# 24. MongoDB Database User

The application uses a MongoDB database user.

The user should have only the permissions required by the application.

Avoid using administrative database credentials for normal application operations.

Prefer:

    Application User
        |
        +---- required database
        +---- required permissions

instead of:

    Application
        |
        +---- full database administration

---

# 25. Docker Container Security

The application container should contain only what it needs.

The Dockerfile uses:

    node:22-bookworm-slim

instead of a full operating-system image.

Benefits include:

- smaller image
- fewer unnecessary packages
- smaller attack surface
- faster transfer

The container should not contain:

- Git repository metadata
- `.env`
- local development files
- unnecessary build artifacts

---

# 26. Production Image Principle

The image should represent the application artifact.

Example:

    registry.example.com/application:COMMIT_SHA

The image should not depend on:

- local source files
- developer machines
- local `.env`
- manual modifications

This supports:

    Build once
        |
        v
    Test
        |
        v
    Store image
        |
        +---- DEV
        +---- TEST
        +---- PROD

---

# 27. Immutable Image Tags

Production deployment uses the Git commit SHA.

Example:

    IMAGE_TAG=abc123...

The production server runs:

    registry.example.com/application:abc123...

rather than relying only on:

    latest

Why?

Because `latest` can move.

A commit SHA identifies a specific build artifact.

This makes rollback much safer.

---

# 28. Do Not Trust `latest` for Rollback

Avoid rollback like:

    docker pull application:latest

because `latest` may now point to the newer version.

Instead:

    docker pull application:PREVIOUS_COMMIT_SHA

Then deploy that exact image.

This creates a clear relationship:

    Git Commit
        |
        v
    Docker Image
        |
        v
    Production Deployment

---

# 29. Credential Exposure Incident

During the deployment work, production configuration output containing sensitive values was accidentally exposed during troubleshooting.

The important lesson is:

> Never use commands that dump complete production configuration into chat, tickets, screenshots, or public documentation.

For example, avoid sharing complete output from:

    docker compose config

when that output contains secrets.

Even if the command is useful for debugging, the output must be treated as sensitive.

---

# 30. Response to Credential Exposure

If a real credential has been exposed, the correct response is:

1. Identify the affected credential.
2. Rotate/revoke it.
3. Generate a replacement.
4. Update the production environment.
5. Restart/redeploy the application if required.
6. Verify the application.
7. Check logs/audit history where available.
8. Ensure the old credential is no longer usable.

Do not assume that a secret is safe simply because the exposure happened in a private conversation.

The affected credentials from the incident should be rotated before considering the production security cleanup complete.

No real credential values belong in this public documentation.

---

# 31. Safe Configuration Debugging

Instead of printing:

    MONGODB_URI=mongodb://username:password@...

print only safe information.

Example:

    MONGODB_URI configured: yes
    JWT_SECRET configured: yes
    API_KEY configured: yes

Never print:

    JWT_SECRET=actual-secret
    API_KEY=actual-key
    MONGODB_URI=actual-uri

Logging configuration presence is safer than logging configuration values.

---

# 32. Logging Security

Application logs should never contain:

- Passwords
- JWT secrets
- API keys
- MongoDB passwords
- Authorization headers
- SSH private keys
- Full sensitive user information

Be careful with:

    console.log(req.headers)

because it may expose:

    Authorization: Bearer <token>

Prefer structured and sanitized logging.

Example:

    Authentication failed for user <user-id>

instead of:

    Authorization header = Bearer eyJ....

---

# 33. Error Response Security

Production API responses should not expose internal implementation details.

Avoid returning:

    MongoServerSelectionError:
    connection string:
    internal file path:
    stack trace:

to clients.

The client should receive a safe error such as:

    {
      "message": "Internal server error"
    }

Detailed information should remain in protected server logs.

---

# 34. Dependency Security

Node.js dependencies should be reviewed regularly.

Useful commands include:

    npm audit

and:

    npm outdated

Do not blindly update every dependency in production.

A dependency update can introduce:

- breaking changes
- runtime issues
- security fixes
- API changes

Use controlled updates and test before deployment.

---

# 35. Package Lock

The project uses:

    package-lock.json

The Docker build uses:

    npm ci --omit=dev

`npm ci` installs from the lock file and provides a more deterministic installation than a general:

    npm install

This helps make CI and production builds reproducible.

---

# 36. Production Dependencies

The production Docker image should install only required runtime dependencies.

The Dockerfile uses:

    npm ci --omit=dev

This avoids installing development dependencies such as:

- nodemon
- Jest
- other development-only packages

This reduces the production image footprint.

---

# 37. SSH Security

EC2 SSH access should be hardened.

Recommended controls include:

- SSH key authentication
- Disable password authentication
- Do not use root login
- Keep Ubuntu packages updated
- Restrict port 22 where practical
- Use a dedicated deployment user in a mature setup
- Remove unused users/keys

The current deployment uses the Ubuntu user.

Further SSH hardening is part of the remaining security work.

---

# 38. Docker Group Privilege

The Ubuntu user was added to the Docker group so Docker commands could run without:

    sudo

This is convenient, but membership in the Docker group provides very high privileges on the host.

Treat Docker group membership as privileged access.

Only trusted users should have it.

Do not add arbitrary users to:

    docker

group.

---

# 39. AWS Security Group Principle

Security Groups should follow:

> Deny by default, allow only required traffic.

Example conceptual configuration:

    80    -> Internet
    443   -> Internet
    22    -> restricted administration source

Avoid:

    3000  -> Internet

when Nginx is the public entry point.

Avoid opening database ports publicly unless specifically required.

---

# 40. Nginx Security

Nginx should be the public application boundary.

Responsibilities include:

- HTTPS
- Domain routing
- Reverse proxy
- WebSocket forwarding
- Request handling

Future hardening can include:

- Security headers
- Request size limits
- Rate limiting
- Connection limits
- Access logging
- Error logging
- More restrictive proxy configuration

These should be introduced based on application requirements.

---

# 41. Rate Limiting

Authentication endpoints are common targets for abuse.

Examples:

    /login
    /register
    /password-reset

A future production hardening step is to introduce rate limiting.

Conceptually:

    Client
       |
       v
    Rate Limit
       |
       v
    Authentication API

This helps reduce brute-force and abusive request traffic.

Rate limits should be designed based on real application behavior.

---

# 42. File Upload Security

The application uses file upload functionality.

File uploads should be treated as untrusted input.

Security controls should include:

- Validate file type
- Validate file size
- Avoid trusting client-provided MIME type alone
- Sanitize file names
- Store uploads outside executable paths
- Restrict accepted formats
- Avoid allowing arbitrary executable content

The exact implementation depends on the application's upload requirements.

---

# 43. Input Validation

All external input should be considered untrusted.

Examples:

    req.body
    req.params
    req.query
    uploaded files
    WebSocket messages

Validate:

- Required fields
- Types
- Lengths
- Allowed values
- Formats
- Authorization

Do not assume that frontend validation is sufficient.

Frontend validation improves UX.

Backend validation provides the security boundary.

---

# 44. WebSocket Input

The application uses WebSocket messages such as:

    start
    audio_data
    respond
    stop
    mode

WebSocket messages must be validated.

Do not assume the client will always send valid messages.

Validate:

    message type
    payload structure
    authenticated user
    session ownership
    message size

Unexpected messages should be rejected safely.

---

# 45. External API Keys

The application uses external services.

Examples include:

- AssemblyAI
- Groq
- Anthropic
- Other AI/service providers

Their API keys must remain server-side.

Do not expose provider keys to:

- React frontend
- Electron client
- Browser JavaScript
- Git repository
- public documentation

Preferred flow:

    Client
       |
       v
    Backend
       |
       +---- External API
              |
              v
           Provider

---

# 46. Production Environment Separation

Do not reuse production credentials in development.

Conceptually:

    DEV
      |
      +---- DEV database
      +---- DEV secrets
      +---- DEV API keys

    TEST
      |
      +---- TEST database
      +---- TEST secrets
      +---- TEST API keys

    PROD
      |
      +---- PROD database
      +---- PROD secrets
      +---- PROD API keys

The same application artifact can be promoted while configuration remains environment-specific.

---

# 47. Build Once, Configure at Runtime

Security and deployment become easier when:

    Source Code
         |
         v
    Build
         |
         v
    Immutable Image
         |
         +------------+
         |            |
        DEV          PROD
         |            |
      Config A      Config B

The image remains the same.

Only runtime configuration changes.

---

# 48. Public GitHub Documentation Security

This portfolio repository is public.

Therefore it must never contain:

- Real EC2 IP addresses
- Real domain names
- Real usernames
- Real email addresses
- MongoDB URI
- MongoDB passwords
- JWT secrets
- Encryption keys
- API keys
- GitLab tokens
- SSH private keys
- Production `.env`
- Real internal hostnames
- Sensitive screenshots

Use placeholders:

    YOUR_EC2_IP
    YOUR_DOMAIN
    YOUR_USERNAME
    YOUR_EMAIL
    YOUR_MONGODB_URI
    YOUR_SECRET
    YOUR_API_KEY
    REGISTRY_IMAGE

---

# 49. Public Repository Example

Safe:

    ssh ubuntu@YOUR_EC2_IP

Safe:

    https://api.YOUR_DOMAIN

Safe:

    registry.gitlab.com/YOUR_NAMESPACE/YOUR_PROJECT

Unsafe:

    ssh ubuntu@REAL_PRODUCTION_IP

Unsafe:

    mongodb+srv://real-user:real-password@...

The purpose of the public repository is to teach the deployment process, not expose the actual infrastructure.

---

# 50. Production Security Checklist

Before calling the deployment secure enough for the current stage:

### Repository

- [x] `.env` ignored
- [x] Secrets not committed
- [x] SSH private key not committed
- [x] Registry credentials not committed
- [x] Public documentation sanitized

### Docker

- [x] `.env` excluded from image
- [x] `.git` excluded from image
- [x] Production dependencies only
- [x] Slim base image
- [x] Immutable image tags

### GitLab

- [x] Registry authentication uses CI variables
- [x] Repository and Registry credentials separated
- [x] Limited PAT scopes used
- [x] Deployment key stored as CI variable

### AWS

- [x] EC2 deployed
- [x] Elastic IP configured
- [x] Security Group configured
- [ ] Public port 3000 removed
- [ ] SSH access hardened

### Nginx

- [x] Reverse proxy configured
- [x] HTTPS enabled
- [x] WebSocket upgrade configured
- [x] Domain configured

### Application

- [x] JWT authentication
- [x] Environment-based secrets
- [x] Production HTTPS
- [ ] Review CORS configuration
- [ ] Review WebSocket validation
- [ ] Add rate limiting where required
- [ ] Review file-upload security

### MongoDB

- [x] MongoDB Atlas used
- [x] EC2 network access configured
- [ ] Review least-privilege database permissions
- [ ] Verify backup/recovery strategy

### Credentials

- [ ] Rotate credentials exposed during troubleshooting
- [ ] Verify old credentials are revoked
- [ ] Review all production secrets

---

# 51. Security Is Continuous

Security is not a one-time phase.

The deployment should be periodically reviewed for:

    Credentials
    Dependencies
    OS updates
    Docker images
    Nginx configuration
    SSH access
    AWS Security Groups
    Database access
    API permissions
    Logs
    Backups

A secure deployment today can become insecure later if dependencies, credentials, infrastructure, or application behavior change.

---

# 52. What Was Actually Learned

This deployment demonstrated several important security principles.

### Principle 1 — Never commit secrets

Environment-specific secrets belong outside Git.

### Principle 2 — Least privilege

Credentials should have only the permissions required.

### Principle 3 — Separate credentials

Git repository access and Container Registry access do not need the same token.

### Principle 4 — Protect the production boundary

Nginx + HTTPS provides the public entry point.

### Principle 5 — Reduce attack surface

Do not expose unnecessary ports.

### Principle 6 — Immutable artifacts

Commit SHA image tags make deployments and rollback traceable.

### Principle 7 — Treat logs as sensitive

Debug output can accidentally expose credentials.

### Principle 8 — Rotate exposed credentials

Once a real secret is exposed, rotation is the correct response.

---

# 53. Current Security Status

Completed:

- Environment-based secrets
- `.env` excluded from Git
- `.env` excluded from Docker image
- GitLab CI/CD variables
- Registry credential separation
- Limited PAT scopes
- SSH deployment
- Nginx reverse proxy
- HTTPS
- JWT authentication
- MongoDB Atlas network access
- Commit-SHA image deployment
- Public repository sanitization rules

Remaining:

- Rotate credentials exposed during troubleshooting
- Remove unnecessary public port 3000
- Harden SSH
- Improve SSH CI variable handling
- Review CORS
- Review WebSocket validation
- Add rate limiting where required
- Review upload security
- Review MongoDB permissions
- Add monitoring and alerting
- Add backup/recovery validation

---

# 54. Verification

The production request path should be:

    Client
      |
      | HTTPS
      v
    Nginx
      |
      | HTTP localhost
      v
    Docker
      |
      v
    Node.js
      |
      v
    MongoDB Atlas

Verify the API:

    curl.exe https://api.YOUR_DOMAIN/health

Expected conceptual response:

    {
      "status": "ok"
    }

Verify Docker:

    docker compose ps

Verify logs:

    docker compose logs --tail=100

Never paste sensitive log output into public documentation.

---

# 55. Security Architecture

```text
                    Internet
                       |
                       | HTTPS / WSS
                       v
                +--------------+
                |    Nginx     |
                | TLS / Proxy  |
                +--------------+
                       |
                       | localhost
                       v
                +--------------+
                | Docker       |
                | Node.js API  |
                +--------------+
                       |
              +--------+--------+
              |                 |
              v                 v
        MongoDB Atlas      External APIs
        Restricted         API Keys
        Access             Server-side only