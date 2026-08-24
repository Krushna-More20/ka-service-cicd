
# ka-service — Backend CI/CD Pipeline (Karmika)

A production-ready CI/CD pipeline for the **Karmika** platform backend service (`ka-service`), built and managed end-to-end. Karmika is a worker and business hiring app for India. This covers everything from local development to automated cloud deployment with manual approval gates.

---

## About Karmika

Karmika is a hiring platform that connects workers and businesses in India. The platform consists of:

| Repo | Purpose |
|---|---|
| `ka-service` | Node.js backend API (this repo) |
| `ka-core` | Core Flutter mobile app |
| `ka-crm-ui` | React CRM/Admin dashboard |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Node.js + Express |
| Containerization | Docker + Docker Compose |
| Container Registry | AWS ECR (Elastic Container Registry) |
| Cloud Server | AWS EC2 (Ubuntu) |
| Reverse Proxy | Nginx |
| CI/CD | GitHub Actions |
| Approval Gate | Manual approval via GitHub Issues |
| Push Notifications | Firebase Cloud Messaging (FCM) |
| Translations | Google Translate API |
| Credentials | AWS IAM + GitHub Secrets |

---

## Project Structure

```
ka-service/
├── src/                              # Express API source code
├── docs/                             # Documentation
├── logs/                             # Log files
├── public/                           # Public assets
├── scripts/                          # Utility scripts
├── secrets/                          # Runtime credentials (git ignored)
├── tests/                            # Jest test files
├── tmp/                              # Temporary files
├── .github/
│   └── workflows/
│       └── ci-cd.yml                 # GitHub Actions pipeline
├── docker-compose.yml                # Dev environment compose
├── docker-compose.prod.yml           # Prod environment compose
├── Dockerfile                        # Docker image definition
├── .dockerignore
├── .env.example
├── firebase.json                     # Firebase config
├── firestore.indexes.json
├── firestore.rules
├── emailcron.js                      # Email cron job
├── notificationCron.js               # Notification cron job
├── server.js                         # App entry point
├── jest.config.js                    # Jest config
└── package.json
```

---

## CI/CD Pipeline Flow

```
Developer pushes code
        ↓
GitHub Actions triggers
        ↓
Checkout code
        ↓
Create runtime credentials
  → Google Translate key JSON
  → Firebase service account JSON
  → Validate Firebase JSON
        ↓
⏳ Manual Approval Gate
  → GitHub Issue auto-created
  → Approver comments "approve"
  → Pipeline continues (timeout: 15 min)
        ↓
Configure AWS Credentials
        ↓
Setup Node.js 20
        ↓
npm install + npm test
        ↓
Login to Amazon ECR
        ↓
Build Docker Image
        ↓
Tag + Push Image to ECR
        ↓
Copy docker-compose to EC2 (via SCP)
        ↓
SSH into EC2
  → Login Docker to ECR
  → docker-compose down
  → docker-compose pull
  → docker-compose up -d
  → docker image prune -f
  → nginx -t && nginx reload
        ↓
Service is live ✅
```

---

## Branch Strategy

| Branch | Environment | Approvers |
|---|---|---|
| `main` | Dev EC2 Server | Krushna or team lead |
| `release` | Prod EC2 Server | Team lead only |

- Push to `main` → triggers **deploy-dev** job → deploys to Dev EC2
- Push to `release` → triggers **deploy-prod** job → deploys to Prod EC2

---

## Manual Approval Gate

Every deployment requires **manual approval** before it proceeds — for both dev and prod.

**How it works:**
1. Developer pushes code to `main` or `release`
2. Pipeline runs credentials setup
3. Pipeline **automatically creates a GitHub Issue** titled:
   - Dev: `"Manual Approval Required For Deployment"`
   - Prod: `"Manual Approval Required For PROD Deployment"`
4. Approver receives notification and reviews the issue
5. Approver comments **"approve"** on the issue
6. Pipeline resumes and deploys
7. If no approval within **15 minutes** → pipeline times out automatically

This prevents any accidental or unauthorized deployments — especially to production.

---

## AWS Setup (Managed End-to-End)

### IAM User Permissions
Configured IAM users with these policies:
```
AmazonEC2FullAccess
AmazonEC2ContainerRegistryFullAccess
IAMReadOnlyAccess
```

Separate IAM credentials used for dev and prod environments.

### ECR Repositories
Created private ECR repositories:
```
339050855802.dkr.ecr.ap-south-1.amazonaws.com/backend-service      ← dev
339050855802.dkr.ecr.ap-south-1.amazonaws.com/ka-prod-service      ← prod
```

ECR Lifecycle Policy keeps only the **last 3 images** — older images are auto-deleted to save storage cost.

### EC2 Instances
- AMI: Ubuntu 22.04
- Region: ap-south-1 (Mumbai)
- Security Group inbound rules:
  - Port 22 (SSH) — restricted to team IPs
  - Port 80 (HTTP) — open to public

---

## EC2 One-Time Setup Commands

After launching EC2, these commands were run once to set up the server:

```bash
# SSH into EC2
ssh -i "your-key.pem" ubuntu@<ec2-ip>

# Update and install all required tools
sudo apt update
sudo apt install -y docker.io docker-compose awscli nginx

# Add ubuntu user to docker group (so pipeline can run docker without sudo)
sudo usermod -aG docker ubuntu

# Re-login to apply group change
exit
ssh -i "your-key.pem" ubuntu@<ec2-ip>

# Verify all tools installed correctly
docker --version
docker-compose --version
nginx -v
aws --version
```

Configure Nginx as reverse proxy:
```bash
sudo nano /etc/nginx/sites-available/default
```

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
# Test and start Nginx
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx

# Create app directories
mkdir -p /opt/ka-dev-01
mkdir -p /opt/ka-prod-01
```

After this one-time setup — **everything else is handled automatically by the pipeline.**

---

## GitHub Secrets

| Secret | Environment | Purpose |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Dev | Dev IAM access key |
| `AWS_SECRET_ACCESS_KEY` | Dev | Dev IAM secret key |
| `PROD_AWS_ACCESS_KEY_ID` | Prod | Prod IAM access key |
| `PROD_AWS_SECRET_ACCESS_KEY` | Prod | Prod IAM secret key |
| `DEV_EC2_HOST` | Dev | Dev EC2 public IP |
| `PROD_EC2_HOST` | Prod | Prod EC2 public IP |
| `EC2_SSH_KEY` | Dev | Dev EC2 private key (.pem) |
| `PROD_EC2_SSH_KEY` | Prod | Prod EC2 private key (.pem) |
| `GH_PAT` | Both | GitHub PAT for approval gate |
| `GOOGLE_TRANSLATE_KEY_JSON` | Both | Google Translate service account |
| `FIREBASE_SERVICE_ACCOUNT_JSON` | Both | Firebase service account for FCM |

---

## ECR Image Cleanup Policy

To keep ECR storage costs low, only the **last 3 images** are retained per repository.

Lifecycle Policy applied on both ECR repos:
```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only last 3 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 3
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

Pipeline also runs `docker image prune -f` on EC2 after every deploy to clean up unused local images.

---

## Run Locally

```bash
# Clone the repo
git clone https://github.com/your-username/ka-service.git
cd ka-service

# Install dependencies
npm install

# Copy env file and fill in values
cp .env.example .env

# Start the server
node server.js
```

---

## Run with Docker Locally

```bash
# Build image
docker build -t ka-service .

# Run container
docker run -p 3000:3000 ka-service
```

---

## Run Tests

```bash
npm test
```

Tests run automatically in the pipeline before every deployment. If tests fail, deployment stops.

---

## What I Built and Managed

- Designed and implemented the full CI/CD pipeline from scratch using GitHub Actions
- Set up separate dev and prod deployment jobs triggered by different branches
- Integrated a **manual approval gate** using GitHub Issues before every deployment
- Configured runtime credential injection for Google Translate and Firebase service accounts
- Set up AWS IAM users with correct minimal permissions for dev and prod
- Created and configured private ECR repositories for both environments
- Launched and configured EC2 instances including Docker, Nginx, and AWS CLI
- Configured Nginx as a reverse proxy on EC2
- Implemented automatic SCP to copy docker-compose files to EC2 on every deploy
- Set up automatic SSH-based deployment with container restart and image cleanup
- Configured ECR Lifecycle Policy to auto-delete old images beyond last 3
- Managed all credentials securely via GitHub Secrets — no hardcoded values anywhere

---

## Author

**Krushna More** — Flutter & Backend Developer
Portfolio: [krushnamore.netlify.app](https://krushnamore.netlify.app)
