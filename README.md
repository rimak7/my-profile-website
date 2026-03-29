# rimadev.com — Portfolio Website

> Personal DevOps portfolio site for **Armande Kouadio** — deployed via a fully automated CI/CD pipeline using GitHub Actions, AWS S3, CloudFront, Route53, SNS, and CloudWatch.

🌐 **Live site:** [www.rimadev.com](https://www.rimadev.com)

---

## Architecture Overview

```
Developer (local)
      │
      │  git push
      ▼
GitHub (main branch)
      │
      │  triggers
      ▼
GitHub Actions CI/CD Pipeline
      │
      ├── 🧪 Test & Lint
      │       ├── Check index.html exists
      │       ├── Validate HTML structure
      │       ├── Check for broken local links
      │       └── Check file size
      │
      ├── 🚀 Deploy (only on push to main)
      │       ├── OIDC → AWS credentials (no long-lived keys)
      │       ├── Sync files to S3
      │       ├── Invalidate CloudFront cache
      │       ├── Wait for invalidation to complete
      │       └── Health check (HTTP 200)
      │
      └── 📣 Notify via SNS
              ├── Email on success
              └── Email on failure
                        │
                        ▼
              www.rimadev.com (live)

— — — between deployments — — —

CloudWatch monitors CloudFront 5xx errors every 5 min
      │
      └── If error rate > 5% → SNS email alert
```

---

## AWS Infrastructure

| Service | Purpose |
|---|---|
| **S3** | Static website hosting — stores `index.html` |
| **CloudFront** | Global CDN — serves site with HTTPS |
| **Route53** | DNS — routes `rimadev.com` → CloudFront |
| **ACM** | SSL/TLS certificate for HTTPS |
| **SNS (FIFO)** | Email notifications on pipeline success/failure |
| **CloudWatch** | Alarm on CloudFront 5xx error rate > 5% |
| **IAM (OIDC)** | Keyless auth between GitHub Actions and AWS |

---

## CI/CD Pipeline

### Trigger
- Every `git push` to `main` branch
- Pull requests to `main` (test job only, no deploy)

### Jobs

#### 1. 🧪 Test & Lint
Runs on every push and PR:
- Verifies `index.html` exists
- Validates HTML structure (`<html>`, `<head>`, `<body>`)
- Checks all local file references exist
- Checks file is not empty (min 500 bytes)

#### 2. 🚀 Deploy to S3 + CloudFront
Runs only on push to `main` after tests pass:
- Authenticates to AWS via **OIDC** — no static credentials stored
- Syncs files to S3 with `--delete` flag to remove stale files
- Creates CloudFront invalidation for `/*`
- Waits for invalidation to complete
- Runs HTTP health check against `https://www.rimadev.com`

#### 3. 📣 Notify via SNS
Runs after deploy (always, even on failure):
- Sends success email with commit SHA, author, branch, and run link
- Sends failure email identifying which job failed with logs link

---

## Security

- **OIDC authentication** — GitHub Actions assumes an IAM role directly, no AWS access keys stored anywhere
- **Least-privilege IAM role** — scoped to only `S3`, `CloudFront`, and `SNS` permissions
- **Role trust policy** — restricted to `repo:rimak7/my-profile-website` only, no other repo can assume it
- **HTTPS enforced** — via ACM certificate attached to CloudFront
- **S3 not directly accessible** — all traffic routed through CloudFront

---

## Monitoring & Alerting

| What | How | Alert |
|---|---|---|
| Pipeline success | GitHub Actions + SNS | Email |
| Pipeline failure | GitHub Actions + SNS | Email |
| Site 5xx errors | CloudWatch Alarm | SNS Email |

**CloudWatch Alarm config:**
- Metric: `CloudFront → 5xxErrorRate`
- Period: 5 minutes
- Threshold: > 5%
- Action: SNS notification → email

---

## Repository Structure

```
my-profile-website/
├── .github/
│   └── workflows/
│       └── deploy.yml       # CI/CD pipeline definition
├── index.html               # Portfolio site (HTML + CSS + JS, single file)
└── README.md
```

---

## GitHub Secrets Required

| Secret | Description |
|---|---|
| `AWS_ROLE_ARN` | IAM role ARN for OIDC authentication |
| `S3_BUCKET` | S3 bucket name |
| `CLOUDFRONT_DISTRIBUTION_ID` | CloudFront distribution ID |
| `SNS_TOPIC_ARN` | SNS FIFO topic ARN for notifications |

---

## Local Development with Docker

Run the site locally using Docker Desktop:

```bash
# Option 1 — bind mount (live file editing)
docker run -d -p 8080:80 \
  -v "//$(pwd):/usr/share/nginx/html:ro" \
  nginx:alpine

# Option 2 — build a self-contained image
docker build -t rimadev .
docker run -d -p 8080:80 --name rimadev rimadev

# Visit http://localhost:8080
```

**Dockerfile:**
```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
```

---

## Deployment

Deployment is fully automated. Just push to `main`:

```bash
git add .
git commit -m "your message"
git push origin main
```

GitHub Actions will automatically:
1. Run tests and lint checks
2. Deploy to S3 and invalidate CloudFront cache
3. Run a health check
4. Send an email notification with the result

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | HTML5, CSS3, Vanilla JavaScript |
| Local server | Docker + nginx:alpine |
| Storage | AWS S3 |
| CDN | AWS CloudFront |
| DNS | AWS Route53 |
| SSL | AWS ACM |
| CI/CD | GitHub Actions |
| Auth | AWS IAM OIDC (keyless) |
| Notifications | AWS SNS FIFO |
| Monitoring | AWS CloudWatch |

---

## Author

**Armande Kouadio** — DevOps Engineer  
📧 armande.kanm7@gmail.com  
📞 (929) 636-5601  
🔗 [LinkedIn](https://linkedin.com/in/armande-kouadio-920770323)  
🐙 [GitHub](https://github.com/rimak7)  
🌐 [www.rimadev.com](https://www.rimadev.com)
