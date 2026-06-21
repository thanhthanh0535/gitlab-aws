## Architecture

```
GitLab CI/CD Pipeline
        │
        ├── build         → npm ci + vite build → build/ artifact
        │
        ├── package       → Docker build → push to AWS ECR
        │                   (amazon/aws-cli + docker:dind)
        │
        └── deploy        → Register ECS task definition → Update ECS service
                            (aws ecs register-task-definition + update-service)
```

**AWS Infrastructure:**
- **ECR** — private Docker registry storing versioned app images
- **ECS Cluster** — `cluster-gitlab-1` running the containerized React app
- **ECS Service** — rolling update triggered automatically on every `main` push
- **Nginx** — serves the React build inside the container (`nginx:1.27.3-alpine`)

---

## Pipeline Stages

| Stage | Job | What it does |
|---|---|---|
| `build` | `build_website` | `npm ci` + `vite build`, passes `build/` as artifact to next stage |
| `package` | `build_docker_image` | Builds Docker image tagged with `$CI_COMMIT_SHORT_SHA`, authenticates to ECR via `aws ecr get-login-password`, pushes all tags |
| `deploy` | `ecs_deploy` | Registers new ECS task definition from `aws/gitlab-prod.json`, updates ECS service to trigger rolling deployment |

**Key pipeline details:**
- `VITE_APP_VERSION` is set to `$CI_COMMIT_SHORT_SHA` — every build is traceable to an exact commit
- Docker image tagged with both commit SHA and `latest` — `docker push --all-tags`
- Uses `amazon/aws-cli:2.35.8` as CI image with `docker:27-dind` as a service for Docker-in-Docker builds
- AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`) stored as masked GitLab CI variables
- ECS task definition stored as `aws/gitlab-prod.json` — version-controlled infrastructure config

---

## Tech Stack

| Layer | Tool |
|---|---|
| Frontend | React 18 + Vite 6 |
| Containerization | Docker (`nginx:1.27.3-alpine`) |
| Container Registry | AWS ECR |
| Orchestration | AWS ECS (Fargate) |
| CI/CD | GitLab CI/CD |
| AWS CLI | `amazon/aws-cli:2.35.8` |
| Unit Testing | Vitest |
| E2E Testing | Playwright (Chromium) |

---

## Dockerfile

```dockerfile
FROM nginx:1.27.3-alpine
COPY build /usr/share/nginx/html
```

Minimal production image — copies the Vite build output into Nginx's serve directory. Alpine base keeps the image size small.

---

## Project Structure

```
├── src/                    # React app source
├── tests/                  # Vitest unit tests
├── e2e/                    # Playwright e2e tests
├── aws/
│   └── gitlab-prod.json    # ECS task definition (version-controlled)
├── ci/                     # Supporting CI scripts
├── .gitlab-ci.yml          # Full 3-stage pipeline
├── Dockerfile              # nginx:alpine production image
├── vite.config.js          # Vite + Vitest config
└── package.json
```

---

## CI/CD Variables (GitLab → Settings → CI/CD → Variables)

| Variable | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key (masked) |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key (masked) |
| `AWS_DEFAULT_REGION` | `us-east-1` |
| `DOCKER_REGISTRY` | ECR registry URL (`<account>.dkr.ecr.<region>.amazonaws.com`) |
| `NETLIFY_AUTH_TOKEN` | Used in earlier Netlify stage (if applicable) |

---

## Local Setup

```bash
# Install dependencies
npm install

# Run dev server
npm run dev

# Build for production
npm run build

# Run unit tests
npm test

# Run e2e tests
npm run e2e

# Build Docker image locally
docker build -t gitlab-app-aws .
docker run -p 8080:80 gitlab-app-aws
```

---

## What I Built Beyond the Course

- **Wrote the complete `.gitlab-ci.yml`** — 3 stages, Docker-in-Docker setup, ECR auth, ECS rolling deploy
- **Configured Docker-in-Docker correctly** — `DOCKER_HOST: tcp://docker:2375/` with `docker:27-dind` service and `dnf install -y docker` inside the AWS CLI image
- **Wrote the Dockerfile** — chose `nginx:1.27.3-alpine` as a minimal production base, copied Vite `build/` output directly
- **Version-tagged every image with commit SHA** — `$CI_COMMIT_SHORT_SHA` baked into both the image tag and `VITE_APP_VERSION` env var for full traceability
- **ECS task definition as code** — `aws/gitlab-prod.json` checked into the repo, registered via `aws ecs register-task-definition --cli-input-json` on every deploy
- **Automated rolling deployment** — `aws ecs update-service` triggers ECS to pull the new image and replace running tasks with zero downtime
