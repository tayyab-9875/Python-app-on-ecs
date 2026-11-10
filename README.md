# Python App on AWS ECS — CI/CD with GitHub Actions

![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-containerized-2496ED?logo=docker&logoColor=white)
![AWS ECS](https://img.shields.io/badge/AWS-ECS%20Fargate-FF9900?logo=amazonecs&logoColor=white)
![AWS ECR](https://img.shields.io/badge/AWS-ECR-FF9900?logo=amazonaws&logoColor=white)

A containerized Python web application that is automatically tested, built, pushed to **Amazon ECR**, and deployed to **Amazon ECS** on every push to `main` — fully driven by a **GitHub Actions** pipeline.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [CI/CD Pipeline](#cicd-pipeline)
- [Run Locally](#run-locally)
- [AWS Setup](#aws-setup)
- [GitHub Configuration](#github-configuration)
- [Deployment](#deployment)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Future Improvements](#future-improvements)
- [Author](#author)

---

## Overview

This project demonstrates an end-to-end, production-style deployment workflow for a containerized application on AWS:

- A lightweight Python web app (`app/app.py`) with unit tests
- A `Dockerfile` that packages the app into a portable image
- A GitHub Actions workflow that runs tests, builds the image, pushes it to ECR, and updates the ECS service
- An ECS task definition (`.aws/task-definition.json`) kept in version control

**Result:** a `git push` to `main` takes code from commit to a running container on ECS with no manual steps.

## Architecture

```
┌──────────────┐    push     ┌──────────────────────────────────────────────┐
│  Developer   │ ──────────► │               GitHub Actions                 │
└──────────────┘             │  1. Install deps & run tests                 │
                             │  2. Authenticate to AWS                      │
                             │  3. Build Docker image                       │
                             │  4. Push image to Amazon ECR                 │
                             │  5. Render task definition with new image    │
                             │  6. Deploy to Amazon ECS                     │
                             └──────────────────────┬───────────────────────┘
                                                    │
                          ┌─────────────────────────┴─────────────┐
                          ▼                                       ▼
                 ┌─────────────────┐                    ┌───────────────────┐
                 │   Amazon ECR    │ ── image pulled ─► │    Amazon ECS     │
                 │ (image registry)│                    │ (cluster/service) │
                 └─────────────────┘                    └─────────┬─────────┘
                                                                  ▼
                                                        Running container
                                                        (public access)
```

## Tech Stack

| Layer              | Technology                          |
| ------------------ | ----------------------------------- |
| Application        | Python                              |
| Testing            | pytest (`requirements-dev.txt`)     |
| Containerization   | Docker                              |
| Container Registry | Amazon ECR                          |
| Orchestration      | Amazon ECS                          |
| CI/CD              | GitHub Actions                      |
| Cloud Provider     | AWS                                 |

## Project Structure

```
Python-app-on-ecs/
├── .aws/
│   └── task-definition.json    # ECS task definition (image updated by pipeline)
├── .github/
│   └── workflows/
│       └── ci.yml              # CI/CD pipeline
├── app/
│   ├── app.py                  # Application source
│   ├── test_app.py             # Unit tests
│   ├── requirements.txt        # Runtime dependencies
│   └── requirements-dev.txt    # Development/test dependencies
├── .dockerignore
├── .gitignore
├── Dockerfile                  # Container image definition
└── README.md
```

## CI/CD Pipeline

The workflow in [`.github/workflows/ci.yml`](.github/workflows/ci.yml) runs on every push to `main`.

| Stage        | Description                                                            |
| ------------ | ---------------------------------------------------------------------- |
| **Test**     | Installs dependencies and runs the unit test suite                     |
| **Build**    | Builds the Docker image, tagged with the commit SHA                    |
| **Push**     | Authenticates with ECR and pushes the image                            |
| **Deploy**   | Injects the new image into the task definition and updates the ECS service, waiting for service stability |

A failing test stops the pipeline, so broken code never reaches ECS.

## Run Locally

**Prerequisites:** Python 3.x, Docker (optional)

### Without Docker

```bash
git clone https://github.com/tayyab-9875/Python-app-on-ecs.git
cd Python-app-on-ecs/app

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
python app.py
```

### With Docker

```bash
docker build -t python-app-on-ecs .
docker run -p 5000:5000 python-app-on-ecs
```

Then open `http://localhost:5000` (adjust the port if your app uses a different one).

## AWS Setup

The following AWS resources are required before the pipeline can deploy:

1. **Amazon ECR repository** — stores the Docker images
2. **Amazon ECS cluster** — hosts the service
3. **ECS task definition** — registered from `.aws/task-definition.json`
4. **ECS service** — runs and maintains the desired number of tasks
5. **IAM role/user for GitHub Actions** — with permissions to push to ECR and update the ECS service
6. **Security group** — allows inbound traffic on the application port

> Replace the placeholder values in `.aws/task-definition.json` (account ID, region, role ARNs, image) with your own.

## GitHub Configuration

Set these under **Settings → Secrets and variables → Actions**.

**Secrets**

| Name                    | Description                      |
| ----------------------- | -------------------------------- |
| `AWS_ACCESS_KEY_ID`     | IAM access key for the pipeline  |
| `AWS_SECRET_ACCESS_KEY` | IAM secret key for the pipeline  |

**Environment variables** (defined at the top of `ci.yml`)

| Name                | Example           |
| ------------------- | ----------------- |
| `AWS_REGION`        | `us-east-1`       |
| `ECR_REPOSITORY`    | `<your-ecr-repo>` |
| `ECS_CLUSTER`       | `<your-cluster>`  |
| `ECS_SERVICE`       | `<your-service>`  |
| `CONTAINER_NAME`    | `<container-name-in-task-definition>` |

> If your workflow authenticates via GitHub OIDC instead of access keys, replace the secrets above with the IAM role ARN to assume.

## Deployment

Deployment is fully automated:

```bash
git add .
git commit -m "Update greeting message"
git push origin main
```

Then follow progress in the repository's **Actions** tab. After a successful run:

1. The new image appears in **Amazon ECR**, tagged with the commit SHA.
2. ECS starts a new task with the updated image and drains the old one.
3. The change is live at the service's public IP / load balancer DNS.

## Testing

```bash
cd app
pip install -r requirements-dev.txt
pytest
```

## Troubleshooting

| Problem                                   | Likely cause / fix                                                                 |
| ----------------------------------------- | ---------------------------------------------------------------------------------- |
| `AccessDeniedException` in the pipeline   | IAM permissions missing for ECR push or `ecs:UpdateService` / `iam:PassRole`       |
| ECS service never becomes stable          | Check task logs in CloudWatch; verify container port and security group rules      |
| Image pull error in ECS                   | Task execution role lacks ECR read permissions, or the image tag does not exist    |
| App unreachable after deploy              | Confirm the security group allows the app port and the task has a public IP        |

## Future Improvements

- [ ] Replace static AWS keys with GitHub OIDC role assumption
- [ ] Add an Application Load Balancer with health checks
- [ ] Add image vulnerability scanning (e.g., Trivy)
- [ ] Provision infrastructure with Terraform
- [ ] Add staging/production environments with manual approval
- [ ] Add CloudWatch alarms and dashboards

## Author

**Muhammad Tayyab** — DevOps Engineer

- GitHub: [@tayyab-9875](https://github.com/tayyab-9875)
- LinkedIn: [muhammad-tayyab](https://www.linkedin.com/in/muhammad-tayyab-0bab48264)
- Email: m.tayyab9875@gmail.com
