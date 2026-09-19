# Production Deployment with Docker, GitLab CI/CD and AWS

A practical, production-oriented deployment workflow covering:

- Local development
- Git
- Docker
- Container Registry
- GitLab CI/CD
- AWS EC2
- Docker Compose
- Nginx
- Domain configuration
- HTTPS
- Production deployment
- Rollback
- Security
- Monitoring

This repository documents a real-world engineering deployment journey in a generalized and sanitized form.

> The actual application repository remains private.  
> This repository contains architecture, processes, examples, commands, decisions, troubleshooting notes, and lessons learned.

---

# Architecture

The baseline production architecture is:

```text
                         Internet
                            |
                            v
                     Domain / DNS
                            |
                            v
                       HTTPS / WSS
                            |
                            v
                         Nginx
                            |
                            v
                    Docker / Node.js
                       /         \
                      /           \
                     v             v
             MongoDB Atlas    External APIs





##############################################################

production-deployment-docker-cicd/
│
├── README.md
├── .gitignore
│
├── application/
│   ├── 00-project-roadmap.md
│   ├── phase-01-local-development.md
│   ├── phase-02-git.md
│   ├── phase-03-docker.md
│   ├── phase-04-container-registry.md
│   ├── phase-05-gitlab-cicd.md
│   ├── phase-06-aws-ec2.md
│   ├── phase-07-nginx.md
│   ├── phase-08-domain-https.md
│   ├── phase-09-production-deployment.md
│   ├── phase-10-rollback.md
│   ├── phase-11-security.md
│   ├── phase-12-monitoring.md
│   └── 99-pending-work.md
│
├── architecture/
│   ├── deployment-flow.md
│   └── production-architecture.md
│
└── examples/
    ├── Dockerfile
    ├── docker-compose.yml
    └── .gitlab-ci.yml