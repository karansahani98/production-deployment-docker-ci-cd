# Contributing

Thanks for your interest in this repo. It documents a real production deployment workflow (Docker, GitLab CI/CD, AWS EC2, Nginx, HTTPS), sanitized for general use as a reference and learning resource.

## What this repo is (and isn't)

- ✅ A documented, phased playbook for deploying a containerized app to production
- ✅ A place to improve clarity, fix errors, and add battle-tested tips
- ❌ Not the application's source code (that repo stays private)
- ❌ Not a place for unrelated tooling, frameworks, or stack rewrites

## Ways to contribute

- **Fix errors** — typos, broken commands, outdated flags/versions in any `phase-*.md` file
- **Improve clarity** — reword confusing steps, add missing context
- **Add troubleshooting notes** — if you hit an issue following a phase, document the fix in the relevant phase file or in `99-pending-work.md`
- **Extend examples** — improvements to `examples/Dockerfile`, `examples/docker-compose.yml`, or `examples/.gitlab-ci.yml` (keep them generic/sanitized — no real credentials, domains, or infra details)
- **Add a new phase** — if there's a production concern not yet covered (e.g. backups, autoscaling, logging), propose it as a new `phase-XX-*.md` file

## How to contribute

1. Fork the repo and create a branch: `git checkout -b fix/phase-07-nginx-typo`
2. Make your changes, keeping the existing file naming and structure (`phase-NN-topic.md`)
3. Keep examples generic — no real domains, IPs, secrets, or account-specific values
4. Open a pull request with a short description of what changed and why

## Style guidelines

- Use clear, imperative language in steps (e.g. "Create the Dockerfile" not "You would create...")
- Include the actual commands used, in fenced code blocks
- If a step has a common pitfall, add a short "Gotcha" or "Note" callout
- Keep phase files self-contained — link to other phases instead of duplicating content

## Questions

Open an issue if you're unsure whether a change fits, or want to propose a new phase before writing it.