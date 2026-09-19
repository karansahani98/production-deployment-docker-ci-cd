# Pending Work — Production Deployment

This file contains the remaining work that was identified during the real production deployment journey.

These items are intentionally documented but **not implemented yet**.

When implementation starts later, update the status of each item here.

---

# Current Project Status

The following deployment foundation has already been completed:

- Local application setup
- Git repository
- Dockerization
- GitLab Container Registry
- GitLab CI/CD
- AWS EC2
- Docker Compose on EC2
- MongoDB Atlas
- Nginx reverse proxy
- Domain configuration
- HTTPS
- Production deployment
- Commit-SHA image tagging
- Basic rollback strategy
- Security documentation
- Monitoring documentation

The remaining items below are future implementation work.

---

# Priority 1 — Production Safety

These should be handled before considering the production deployment fully hardened.

---

## 1. Rotate Exposed Production Credentials

### Status

    PENDING

### Reason

During troubleshooting, production configuration output containing sensitive credentials was accidentally exposed.

The actual credentials must not be stored in this documentation.

### Steps Later

1. Identify every credential that was exposed.
2. Revoke or rotate each credential.
3. Generate replacement credentials.
4. Update the production `.env`.
5. Restart/redeploy the application if required.
6. Verify application functionality.
7. Confirm the old credentials no longer work.
8. Review logs/audit information where available.
9. Confirm no credential exists in Git history or public documentation.

### Important

Never put replacement credentials into this repository.

---

## 2. Remove Public Port 3000

### Status

    PENDING

### Current Architecture

    Internet
        |
        v
    Nginx :443
        |
        v
    Node.js :3000

Port `3000` is an internal application port.

### Steps Later

1. Open AWS EC2 Security Group.
2. Find inbound rule for TCP port `3000`.
3. Remove public access.
4. Keep required public ports:
   - `80`
   - `443`
5. Keep SSH access only as required.
6. Verify:

       https://api.YOUR_DOMAIN/health

7. Verify the application still works through Nginx.
8. Confirm direct public access to port `3000` is no longer available.

### Goal

    Internet
       |
       v
    Nginx :443
       |
       v
    localhost:3000

---

## 3. SSH Hardening

### Status

    PENDING

### Current

EC2 is accessed using SSH keys.

### Steps Later

1. Review current SSH configuration.
2. Confirm root login is disabled.
3. Confirm password authentication is disabled.
4. Review authorized SSH keys.
5. Remove unused keys.
6. Restrict SSH access where practical.
7. Consider allowing SSH only from trusted administration IPs.
8. Verify CI/CD deployment access still works.
9. Test manual SSH access after changes.

### Goal

Reduce unnecessary SSH attack surface without breaking deployment.

---

## 4. Improve GitLab SSH Key Handling

### Status

    PENDING

### Current

The CI/CD pipeline stores the SSH private key as a GitLab CI/CD variable.

### Improvement

Use a GitLab **File-type CI/CD variable** for the private key.

### Steps Later

1. Create a File-type variable.
2. Store the deployment private key securely.
3. Update `.gitlab-ci.yml`.
4. Remove the old multiline key variable.
5. Run a pipeline.
6. Verify GitLab can connect to EC2.
7. Verify deployment still works.
8. Remove the old variable after successful verification.

---

# Priority 2 — CI/CD Improvements

---

## 5. Add Proper Automated Tests to CI

### Status

    PENDING

### Current Situation

The application contains:

- Unit tests
- Integration tests
- E2E tests

However, the current CI pipeline does not yet execute the complete test suite as part of the production deployment flow.

### Why

The initial attempt showed that:

- Some tests expected a running application.
- Some tests expected database access.
- Some environment variables were required.
- Integration/E2E tests require additional test infrastructure.

Therefore this work was intentionally deferred.

### Steps Later

1. Separate test types clearly.
2. Define required test environment variables.
3. Create a dedicated test database.
4. Start the application when required.
5. Run unit tests.
6. Run integration tests.
7. Run E2E tests.
8. Make CI fail when tests fail.
9. Keep production secrets completely separate.
10. Decide whether deployment should happen only after all tests pass.

### Desired Flow

    Git Push
       |
       v
    Build
       |
       v
    Unit Tests
       |
       v
    Integration Tests
       |
       v
    E2E Tests
       |
       v
    Docker Image
       |
       v
    Registry
       |
       v
    Manual Production Deploy

---

## 6. Add Deployment Health Check

### Status

    PENDING

### Current

Deployment currently verifies the Docker Compose state.

### Future

The deployment should verify the actual application health.

### Steps Later

1. Deploy the new image.
2. Wait for application startup.
3. Call the health endpoint.
4. Verify HTTP status `200`.
5. Verify expected response.
6. Fail the deployment if health verification fails.

### Desired Flow

    docker compose up -d
            |
            v
       Health Check
        /       \
      PASS      FAIL
       |          |
       v          v
   Success     Failure

---

## 7. Automatic Rollback on Failed Health Check

### Status

    PENDING

### Current

Rollback is documented and can be performed manually.

### Future

CI/CD should be able to detect a failed deployment and restore the previous image.

### Steps Later

1. Record the currently deployed image.
2. Deploy the new commit-SHA image.
3. Run health check.
4. If health succeeds:
   - Mark deployment successful.
5. If health fails:
   - Stop/replace failed version.
   - Restore previous commit-SHA image.
   - Start previous version.
   - Run health check again.
6. Mark deployment failed.
7. Report rollback result.

### Desired Flow

    Version A
       |
       v
    Deploy B
       |
       v
    Health Check
       |
    +--+--+
    |     |
   PASS  FAIL
    |     |
    v     v
 Success Rollback A
          |
          v
      Health Check

---

# Priority 3 — Environment Strategy

---

## 8. Define DEV / TEST / PROD Strategy

### Status

    PENDING

### Current

The production deployment is working.

The environment strategy has not yet been fully implemented.

### Future Concept

    DEV
     |
     v
    TEST
     |
     v
    PROD

The exact infrastructure can be decided later.

### Possible Initial Architecture

Because the project is cost-sensitive, separate logical environments may initially run on the same EC2 infrastructure.

Example:

    EC2
     |
     +---- DEV
     |
     +---- TEST
     |
     +---- PROD

Later, environments can move to separate infrastructure if required.

### Steps Later

1. Define environment purpose.
2. Define environment-specific configuration.
3. Define environment-specific database strategy.
4. Define environment-specific secrets.
5. Define deployment rules.
6. Define branch strategy.
7. Define CI/CD variables.
8. Define promotion flow.
9. Define access permissions.

---

# Priority 4 — Monitoring & Operations

---

## 9. Implement External Uptime Monitoring

### Status

    PENDING

### Current

The `/health` endpoint exists and can be checked manually.

### Future

An external monitoring service should periodically call:

    https://api.YOUR_DOMAIN/health

### Steps Later

1. Select uptime monitoring service.
2. Configure health URL.
3. Configure monitoring interval.
4. Configure timeout.
5. Configure failure threshold.
6. Configure notification channel.
7. Test failure alert.
8. Test recovery notification.

### Goal

Detect outages without depending on the EC2 server itself.

---

## 10. Add Monitoring Alerts

### Status

    PENDING

### Metrics

Monitor:

- API availability
- HTTP errors
- CPU
- Memory
- Disk
- Container restarts
- SSL certificate expiry
- Application errors

### Steps Later

1. Define important metrics.
2. Define thresholds.
3. Configure monitoring.
4. Configure notifications.
5. Test alerts.
6. Reduce alert noise.
7. Document incident response.

---

## 11. Add Request Latency Metrics

### Status

    PENDING

### Goal

Track how long API requests take.

### Future Metrics

    Request count
    Request latency
    HTTP 4xx
    HTTP 5xx
    Error rate

### Steps Later

1. Add request timing.
2. Store safe metrics.
3. Track endpoint-level performance.
4. Identify slow endpoints.
5. Establish baseline.
6. Add alerts for abnormal latency if required.

---

## 12. Add Centralized Logging if Required

### Status

    PENDING

### Current

Logs are available through:

    docker compose logs

and:

    /var/log/nginx/

### Future

Centralized logging can be introduced if the application grows.

### Steps Later

1. Define logging requirements.
2. Choose logging platform.
3. Configure log shipping.
4. Protect log access.
5. Define retention.
6. Remove sensitive data from logs.
7. Add search/filtering.
8. Test incident investigation.

### Important

Do not introduce centralized logging just for the sake of adding infrastructure.

Use it when the operational need justifies it.

---

# Priority 5 — Database Reliability

---

## 13. MongoDB Backup & Recovery

### Status

    PENDING

### Current

Production database:

    MongoDB Atlas

### Future

A documented backup and recovery strategy is required.

### Steps Later

1. Review current Atlas backup capabilities.
2. Define backup frequency.
3. Define retention.
4. Define recovery procedure.
5. Document restore process.
6. Test restoring data.
7. Record recovery time.
8. Record recovery point.
9. Document emergency recovery procedure.

### Important

A backup is not considered reliable until restoration has been tested.

---

# Priority 6 — Application Security Review

---

## 14. Review CORS Configuration

### Status

    PENDING

### Steps Later

1. Review allowed origins.
2. Remove unnecessary origins.
3. Verify production frontend/client requirements.
4. Test authenticated requests.
5. Test unauthorized origins.
6. Document final configuration.

---

## 15. Review WebSocket Security

### Status

    PENDING

### Areas

Review:

- Authentication
- Authorization
- Session ownership
- Message validation
- Payload size
- Connection limits
- Disconnect handling
- Error handling
- External streaming service failures

### Steps Later

1. Review WebSocket authentication.
2. Review session ownership.
3. Validate every message type.
4. Validate payload size.
5. Test malformed messages.
6. Test unauthorized connections.
7. Test connection cleanup.
8. Test high connection counts.

---

## 16. Add API Rate Limiting

### Status

    PENDING

### Priority Areas

Especially review:

    /login
    /register
    password-related endpoints
    expensive AI endpoints

### Steps Later

1. Identify abuse-sensitive endpoints.
2. Define limits.
3. Add rate-limiting middleware.
4. Configure production values.
5. Test normal traffic.
6. Test excessive traffic.
7. Monitor false positives.

---

## 17. Review File Upload Security

### Status

    PENDING

### Areas

Review:

- File type
- File size
- MIME validation
- File name handling
- Storage location
- Executable content
- Malicious files
- Upload authorization

### Steps Later

1. Identify every upload endpoint.
2. Define allowed formats.
3. Define maximum size.
4. Validate files server-side.
5. Sanitize names.
6. Test malicious inputs.
7. Review storage permissions.

---

## 18. Review MongoDB Permissions

### Status

    PENDING

### Goal

Ensure the application database user has only the permissions required by the application.

### Steps Later

1. Review current database role.
2. Identify required operations.
3. Define minimum required permissions.
4. Create/modify application user.
5. Test application.
6. Verify administrative operations are unavailable to application credentials.

---

# Priority 7 — Infrastructure Improvements

---

## 19. Review AWS Security Group

### Status

    PENDING

### Final Expected Public Access

    80
    443

SSH:

    22

should be restricted as much as practical.

Application:

    3000

should not be publicly exposed.

### Steps Later

1. Review all inbound rules.
2. Remove unused rules.
3. Remove public port 3000.
4. Restrict SSH where practical.
5. Review outbound rules if required.
6. Document final rules.

---

## 20. Review Docker Image Retention

### Status

    PENDING

### Reason

Every commit creates an image:

    IMAGE_TAG = CI_COMMIT_SHA

Over time, many images may accumulate.

However, old images are useful for rollback.

### Steps Later

1. Define how many versions should be retained.
2. Keep recent production versions.
3. Keep rollback candidates.
4. Remove unnecessary images.
5. Automate cleanup carefully.
6. Never delete the currently deployed image.

---

# Priority 8 — CI/CD Improvements

---

## 21. Define Production Deployment Approval

### Status

    PENDING

### Current

Production deployment is manually triggered from GitLab CI/CD.

### Future

Document:

- Who can deploy
- When deployment can happen
- Approval requirements
- Rollback authority
- Emergency deployment process

The exact process can evolve as the team grows.

---

## 22. Add Deployment Audit Information

### Status

    PENDING

### Goal

Every production deployment should answer:

    Which commit?
    Which image?
    When?
    Who triggered it?
    Was health successful?
    Was rollback required?

The commit SHA already provides the foundation.

### Future

Capture deployment metadata in CI/CD.

---

# Priority 9 — Portfolio Documentation

---

## 23. Final Public GitHub Review

### Status

    PENDING

### Repository

    production-deployment-docker-cicd

### Review

Before publishing:

1. Review README.
2. Review roadmap.
3. Review all 12 phases.
4. Review architecture documents.
5. Review example files.
6. Search for secrets.
7. Search for real IP addresses.
8. Search for real domains.
9. Search for real usernames.
10. Search for real email addresses.
11. Search for tokens.
12. Search for passwords.
13. Search for private keys.
14. Confirm placeholders are used.
15. Confirm real application source code is not included.

### Important

The actual production application remains in the private GitLab repository.

The public GitHub repository contains only:

- Generalized process
- Sanitized commands
- Architecture
- Lessons learned
- Example configurations

---

# Later Projects

These are intentionally outside the current backend deployment roadmap.

---

## 24. Public Website Deployment

### Status

    LATER

The public website has a separate lifecycle.

It should not be mixed into the current backend deployment implementation.

Possible responsibilities:

- SEO
- Basic information
- User registration
- Web deployment
- Domain configuration

This will be handled separately.

---

## 25. Electron Application Distribution

### Status

    LATER

The Electron application has a separate distribution/release lifecycle.

It should not be mixed into the backend deployment pipeline unless required.

Future topics may include:

- Build
- Packaging
- Versioning
- Release
- Auto-update
- Distribution

This is intentionally deferred.

---

# Complete Pending Checklist

```text
PRODUCTION SAFETY

[ ] 1. Rotate exposed production credentials
[ ] 2. Remove public port 3000
[ ] 3. Harden SSH
[ ] 4. Improve GitLab SSH key handling


CI/CD

[ ] 5. Add proper automated tests
[ ] 6. Add deployment health check
[ ] 7. Add automatic rollback


ENVIRONMENTS

[ ] 8. Define DEV / TEST / PROD strategy


MONITORING

[ ] 9. External uptime monitoring
[ ] 10. Monitoring alerts
[ ] 11. Request latency metrics
[ ] 12. Centralized logging if required


DATABASE

[ ] 13. MongoDB backup/recovery


APPLICATION SECURITY

[ ] 14. Review CORS
[ ] 15. Review WebSocket security
[ ] 16. Add API rate limiting
[ ] 17. Review file upload security
[ ] 18. Review MongoDB permissions


INFRASTRUCTURE

[ ] 19. Review AWS Security Group
[ ] 20. Review Docker image retention


CI/CD GOVERNANCE

[ ] 21. Define production deployment approval
[ ] 22. Add deployment audit information


PUBLIC PORTFOLIO

[ ] 23. Final public GitHub review


LATER

[ ] 24. Public website deployment
[ ] 25. Electron application distribution