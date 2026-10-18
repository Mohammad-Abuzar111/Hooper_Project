# Hooper_Project
# Multi-Region AWS Production Architecture and CI/CD

> A production AWS setup for a microservices platform serving web and mobile clients, deployed across Mumbai and Singapore, and moved from a public-subnet design to a private-subnet, load-balanced design.

**Walkthrough video:** [add your Loom / YouTube link here]

**Role:** DevOps Engineer, [Company name], [month year] to present
**My contribution:** [One or two lines, only what you personally did. Example: "Redesigned the network layout, set up the internal ALB and target groups, and maintained the Jenkins deployment pipelines."]

---

## 1. Problem

The platform is a microservices backend (about 20 services) used by a web app and a mobile app. The first deployment ran every service in a public subnet in Singapore. That worked, but it exposed application and database servers more than needed and made traffic routing hard to manage.

Goals:

- Keep application and data tiers off the public internet
- Route traffic through load balancers with per-service health checks
- Deploy reliably from GitLab to multiple regions
- Collect logs from every service in one place

## 2. Architecture

**Traffic flow:** Web and mobile clients → Route 53 → public-facing ALB → web tier → internal ALB → services → data tier.

| Layer | What runs there |
|---|---|
| Edge | Route 53 for DNS, public-facing Application Load Balancer |
| Public subnet | Bastion host only (the single admin entry point) |
| Private subnet: web | Web server fronting the application |
| Private subnet: services | API, auth (UAA), gateway, JWT provider, auditor, Firebase sync, location tracking, S3 proxy, SSE, email automation, task engine, scheduler. Each service sits behind its own target group on an internal ALB |
| Private subnet: data | MongoDB replica set (primary and two secondaries), Redis, Elasticsearch, MySQL |
| Managed services | S3 for storage, SES for email, Firebase for push and sync |
| Observability | Graylog for log consolidation |

## 3. What changed: public to private subnets

| | Before (Singapore) | After (Mumbai) |
|---|---|---|
| Servers | Application and database servers in the public subnet | Only the bastion host is public |
| Routing | Services grouped by host | Internal ALB with service-wise target groups |
| Admin access | Direct | Through the bastion host |
| Databases | Reachable inside a flat network | Isolated in the private subnet |

Reference diagram for the earlier design: `architecture-singapore.png`

## 4. CI/CD

- **Source:** GitLab
- **Build and deploy:** Jenkins (hosted in the Singapore region) deploying to both Mumbai and Singapore environments
- **Flow:** commit → pipeline trigger → build → deploy to target servers → health check through the load balancer

A simplified, sanitized example pipeline is in [`examples/Jenkinsfile`](examples/Jenkinsfile). *(Optional: add this file, written with dummy values.)*

## 5. Monitoring and logging

- Application and system logs from every service are shipped to Graylog
- Search, filtering and alerting happen from one place instead of per server

## 6. Results

Add only real numbers you can stand behind:

- Uptime: [value]
- Deployment time: [before] → [after]
- Incidents or downtime reduced: [value]
- Cost impact: [value]

## 7. Tech stack

AWS (EC2, VPC, ALB, Route 53, S3, SES), MongoDB, Redis, Elasticsearch, MySQL, GitLab, Jenkins, Graylog, Firebase, Linux, Bash

## 8. Repository contents

```
.
├── README.md
├── architecture-mumbai.png
├── architecture-singapore.png
└── examples/
    └── Jenkinsfile        (optional, sanitized sample)
```

## Notes

Service names, hostnames, IP addresses and account identifiers have been removed or generalized. This repository describes the architecture and my work on it; it does not contain company source code or credentials.
