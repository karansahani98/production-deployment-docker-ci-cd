# Phase 02 — Git Workflow

## Status

✅ Completed

## Purpose

This phase establishes source-code version control and connects the private application repository to GitLab.

The flow implemented was:

```text
Local Application
       |
       v
Git Repository
       |
       v
GitLab Remote Repository
       |
       v
main branch
```

The actual application repository is private.

This public portfolio repository documents the process only.

---

# 1. Why Git Comes Before CI/CD

CI/CD needs a reliable source of truth.

The deployment flow eventually becomes:

```text
Developer
    |
    | git push
    v
GitLab
    |
    v
GitLab CI/CD
    |
    v
Docker Build
    |
    v
Container Registry
    |
    v
Production
```

Therefore, before creating CI/CD, the application needs:

- Git repository
- Main branch
- Remote repository
- Reliable authentication
- Clean commit history

---

# 2. Repository Root

An important decision was made during implementation:

The backend application directory itself is the Git repository root.

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
└── .gitignore
```

The `.git` directory exists at the backend root.

Therefore Git commands are executed from:

```text
backend/
```

not from its parent directory.

This decision later became important when configuring GitLab CI/CD.

---

# 3. Check Git Installation

Verify Git:

```powershell
git --version
```

Expected format:

```text
git version 2.x.x
```

The exact version depends on the development machine.

---

# 4. Navigate to Application Repository

Use the real private application directory locally.

Generic example:

```powershell
cd C:\path\to\application
```

Do not publish the real local path if it contains private information that is not needed for the portfolio.

---

# 5. Initialize Git

If the application is not already a Git repository:

```powershell
git init
```

This creates:

```text
.git/
```

inside the current directory.

Verify:

```powershell
git status
```

---

# 6. Configure the Main Branch

The project uses:

```text
main
```

as the primary branch.

Set it with:

```powershell
git branch -M main
```

Verify:

```powershell
git branch
```

Expected:

```text
* main
```

---

# 7. Create `.gitignore`

The application must not commit local dependencies, environment files, logs, or private keys.

General example:

```gitignore
node_modules/
.env
*.log
*.pem
*.key
npm-debug.log
.DS_Store
```

The exact `.gitignore` should also include any application-specific generated files that do not belong in source control.

---

# 8. Why `.env` Is Ignored

Environment configuration can contain:

```text
Database credentials
JWT secrets
API keys
Encryption keys
Service credentials
```

Therefore:

```text
.env
```

must remain outside Git.

Verify that Git does not track it:

```powershell
git status
```

If `.env` appears as an untracked file, add it to `.gitignore` before committing.

---

# 9. Review Files Before First Commit

Run:

```powershell
git status
```

Then inspect what will be committed:

```powershell
git status --short
```

Review important files manually before:

```powershell
git add .
```

Do not blindly commit a repository containing:

```text
.env
*.pem
API keys
password files
private certificates
database dumps
large generated files
node_modules
```

---

# 10. Add Files

Once the repository has been reviewed:

```powershell
git add .
```

Check the staged files:

```powershell
git status
```

Git should now show files under:

```text
Changes to be committed
```

---

# 11. First Commit

Create the initial commit:

```powershell
git commit -m "Initial application setup"
```

Verify:

```powershell
git log --oneline -1
```

Expected pattern:

```text
<commit-sha> Initial application setup
```

The exact SHA is unique to the repository.

---

# 12. Create the GitLab Repository

The real application repository was created privately on GitLab.

For public documentation, use:

```text
https://gitlab.com/OWNER/PRIVATE-REPOSITORY.git
```

The repository must remain private because it contains the real application source.

---

# 13. Add GitLab Remote

From the application repository:

```powershell
git remote add origin https://gitlab.com/OWNER/PRIVATE-REPOSITORY.git
```

Verify:

```powershell
git remote -v
```

Expected pattern:

```text
origin  https://gitlab.com/OWNER/PRIVATE-REPOSITORY.git (fetch)
origin  https://gitlab.com/OWNER/PRIVATE-REPOSITORY.git (push)
```

Never publish a real private repository URL if the repository is intended to remain private.

---

# 14. Push Main Branch

Push the local main branch:

```powershell
git push -u origin main
```

The:

```text
-u
```

sets the upstream tracking relationship.

After this, future pushes can normally use:

```powershell
git push
```

---

# 15. Verify Branch Tracking

Run:

```powershell
git branch -vv
```

The local `main` branch should track:

```text
origin/main
```

---

# 16. Verify Remote

Run:

```powershell
git remote -v
```

Then:

```powershell
git ls-remote origin
```

The second command verifies that Git can communicate with the remote repository.

---

# 17. Authentication

GitLab authentication was handled separately from Docker Registry authentication.

There were two different purposes:

```text
Git Repository
       |
       v
Source code read/write
```

and:

```text
Container Registry
       |
       v
Docker image read/write
```

These should not be treated as the same credential by default.

---

# 18. Repository Access Token

For Git repository operations, the authentication credential needs repository access.

General principle:

```text
read_repository
write_repository
```

Use the minimum permissions necessary.

The actual token value must never be stored in this repository.

For public documentation:

```text
GIT_REPOSITORY_TOKEN=YOUR_TOKEN
```

is acceptable only as a placeholder example.

Never put a real token in a Markdown file.

---

# 19. Registry Credential Is Different

The Docker Registry uses a separate authentication mechanism.

Conceptually:

```text
Git Push
   |
   v
GitLab Repository Credential
```

while:

```text
Docker Push
   |
   v
GitLab Container Registry Credential
```

Keeping these responsibilities separate follows the principle of least privilege.

---

# 20. Authentication Problem Encountered

During implementation, Git operations initially encountered stale credentials stored on the Windows machine.

The local credential manager contained old GitLab credentials.

As a result, Git could attempt to authenticate with an incorrect account/token.

The important lesson was:

```text
Correct token
+
Correct remote
+
Correct local credential cache
```

are all required.

---

# 21. Windows Credential Manager

When Git authentication behaves unexpectedly on Windows, inspect:

```text
Windows Credential Manager
```

Look for stored Git credentials associated with:

```text
gitlab.com
```

Remove only the stale GitLab **Git** credentials that are causing the authentication problem.

Do not blindly remove unrelated credentials.

---

# 22. Important Registry Distinction

GitLab repository authentication and Docker Registry authentication use different hosts/concepts.

Git repository:

```text
gitlab.com
```

Docker Registry:

```text
registry.gitlab.com
```

Do not delete registry credentials when troubleshooting Git repository authentication.

This distinction was important during the implementation.

---

# 23. Test Git Authentication

After correcting stale credentials:

```powershell
git ls-remote origin
```

If authentication succeeds, Git returns repository references.

For example:

```text
<sha>    HEAD
<sha>    refs/heads/main
```

The actual SHA values are repository-specific and should not be copied into generic public documentation unless necessary.

---

# 24. Push Again

After authentication is corrected:

```powershell
git push -u origin main
```

Verify:

```powershell
git status
```

A clean repository should report that there are no uncommitted changes.

---

# 25. Final Git Verification

Run:

```powershell
git status
```

Then:

```powershell
git branch -vv
```

Then:

```powershell
git remote -v
```

Then:

```powershell
git log --oneline -5
```

Finally:

```powershell
git ls-remote origin
```

These commands verify:

```text
Working tree
    +
Branch
    +
Remote
    +
Commit history
    +
Remote connectivity
```

---

# 26. Git Workflow After Initial Setup

Once the repository is initialized, the normal workflow becomes:

```text
Make Code Changes
       |
       v
git status
       |
       v
git diff
       |
       v
git add .
       |
       v
git commit
       |
       v
git push
```

Example:

```powershell
git status
git diff
git add .
git commit -m "Update backend configuration"
git push
```

---

# 27. Useful Git Commands

Check status:

```powershell
git status
```

Show changes:

```powershell
git diff
```

Show staged changes:

```powershell
git diff --cached
```

Show commits:

```powershell
git log --oneline
```

Show current branch:

```powershell
git branch
```

Show remote:

```powershell
git remote -v
```

Check remote access:

```powershell
git ls-remote origin
```

---

# 28. Important Repository Rule

Before every commit:

```powershell
git status
```

Look carefully at the files being added.

Especially check for:

```text
.env
credentials
keys
certificates
logs
database dumps
temporary files
node_modules
```

A good Git workflow is not just:

```text
git add .
```

It is:

```text
Review
   ↓
Stage
   ↓
Review again
   ↓
Commit
```

---

# 29. Security Checklist

Before pushing:

```text
[ ] No .env
[ ] No API keys
[ ] No database passwords
[ ] No JWT secrets
[ ] No encryption keys
[ ] No SSH private keys
[ ] No PEM files
[ ] No registry passwords
[ ] No production configuration
[ ] No sensitive logs
```

---

# 30. Phase Result

At the end of this phase:

```text
Local Application
       |
       v
Git Repository
       |
       v
main branch
       |
       v
Private GitLab Repository
```

The repository was successfully connected to GitLab and the main branch was pushed.

---

# 31. Important Engineering Decisions

## Decision 1 — Private Application Repository

The real application repository remains private.

The public portfolio repository documents the deployment process separately.

---

## Decision 2 — Main Branch

The primary branch is:

```text
main
```

This later becomes the branch used by the production CI/CD rules.

---

## Decision 3 — Separate Credentials

Git repository credentials and container registry credentials are treated as separate access mechanisms.

This supports least-privilege access.

---

## Decision 4 — Secrets Never Belong in Git

Environment variables and private credentials are supplied outside source control.

---

## Decision 5 — Verify Before Push

The repository is checked with:

```powershell
git status
```

before commits and pushes.

---

# 32. Troubleshooting Summary

## Git authenticates with the wrong account

Check:

```text
Windows Credential Manager
```

and remove the stale GitLab Git credential.

Then retry:

```powershell
git ls-remote origin
```

---

## Remote is wrong

Check:

```powershell
git remote -v
```

Correct it if necessary:

```powershell
git remote set-url origin https://gitlab.com/OWNER/PRIVATE-REPOSITORY.git
```

---

## Branch is not tracking remote

Run:

```powershell
git push -u origin main
```

---

## Accidentally staged sensitive file

Before committing:

```powershell
git status
```

Remove it from staging:

```powershell
git restore --staged <file>
```

Then add it to `.gitignore`.

If a secret has already been committed or pushed, simply deleting it in a later commit is not sufficient. The credential should be treated as exposed and rotated.

---

# 33. Phase Completion

```text
[✓] Git installed
[✓] Repository initialized
[✓] main branch configured
[✓] .gitignore configured
[✓] Initial commit created
[✓] GitLab remote configured
[✓] Authentication configured
[✓] Stale credentials troubleshot
[✓] Remote connectivity verified
[✓] Main branch pushed
[✓] Working tree verified
```

---

# 34. Next Phase

➡️ **Phase 03 — Docker**

The next phase will document the complete Docker journey:

```text
Dockerfile
     ↓
.dockerignore
     ↓
Docker build
     ↓
Image inspection
     ↓
Container startup
     ↓
MongoDB networking issue
     ↓
host.docker.internal
     ↓
Container verification
     ↓
Production image considerations
```

All real application information and credentials remain private.