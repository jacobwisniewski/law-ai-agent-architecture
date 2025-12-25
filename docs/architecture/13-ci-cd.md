# CI/CD Architecture

## Overview

GitHub Actions for CI/CD, deploying to AWS ECS Fargate. Simple pipeline optimized for solo developer workflow.

## Environments

| Environment | Purpose | Deployment Trigger |
|-------------|---------|-------------------|
| **Local** | Development | Manual (`pnpm dev`) |
| **Production** | Live | Merge to `main` |

No staging environment for MVP - keeps things simple. Add staging later if needed.

## Branch Strategy

```
main (protected)
  │
  ├── feature/add-search
  ├── feature/m365-connector
  ├── fix/acl-caching
  └── ...
```

- **Feature branches** for all work
- **PR required** to merge to `main`
- **`main` always deployable** - merging triggers production deploy

## Pipeline Stages

### On Pull Request

```yaml
PR opened/updated:
  ├── Install dependencies (pnpm)
  ├── Lint (eslint)
  ├── Type check (tsc --noEmit)
  ├── Unit tests (vitest)
  ├── Build check (ensure compilation)
  └── Migration check (dry-run against test DB)
```

All checks must pass before merge.

### On Merge to Main

```yaml
Merge to main:
  ├── All PR checks (lint, type-check, test, build)
  │
  ├── Build & Push
  │   ├── Build Docker image (api)
  │   ├── Build Docker image (web)
  │   └── Push to ECR
  │
  ├── Database
  │   └── Run migrations (kysely-ctl)
  │
  └── Deploy
      ├── Update ECS service (api)
      └── Update ECS service (web) or S3/CloudFront
```

## Docker Images

Two images:

| Image | Contents | Base |
|-------|----------|------|
| `law-ai/api` | Fastify + tRPC + workers | `node:20-slim` |
| `law-ai/web` | Vite build output + nginx | `nginx:alpine` |

### API Dockerfile

```dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY pnpm-lock.yaml package.json pnpm-workspace.yaml ./
COPY apps/api/package.json ./apps/api/
COPY packages/shared/package.json ./packages/shared/
COPY packages/db/package.json ./packages/db/
RUN corepack enable && pnpm install --frozen-lockfile

COPY . .
RUN pnpm --filter @law-ai/api build

FROM node:20-slim
WORKDIR /app
COPY --from=builder /app/apps/api/dist ./dist
COPY --from=builder /app/apps/api/package.json ./
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3001
CMD ["node", "dist/index.js"]
```

### Web Dockerfile

```dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY pnpm-lock.yaml package.json pnpm-workspace.yaml ./
COPY apps/web/package.json ./apps/web/
COPY packages/shared/package.json ./packages/shared/
RUN corepack enable && pnpm install --frozen-lockfile

COPY . .
RUN pnpm --filter @law-ai/web build

FROM nginx:alpine
COPY --from=builder /app/apps/web/dist /usr/share/nginx/html
COPY apps/web/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## GitHub Actions Workflows

### PR Check (`.github/workflows/pr.yml`)

```yaml
name: PR Check

on:
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: 9
          
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
          
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm type-check
      - run: pnpm test
      - run: pnpm build
```

### Deploy (`.github/workflows/deploy.yml`)

```yaml
name: Deploy

on:
  push:
    branches: [main]

env:
  AWS_REGION: ap-southeast-2
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-2.amazonaws.com

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm type-check
      - run: pnpm test
      - run: pnpm build

  build-and-push:
    needs: check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
          
      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2
        
      - name: Build and push API
        run: |
          docker build -f apps/api/Dockerfile -t $ECR_REGISTRY/law-ai-api:${{ github.sha }} .
          docker push $ECR_REGISTRY/law-ai-api:${{ github.sha }}
          
      - name: Build and push Web
        run: |
          docker build -f apps/web/Dockerfile -t $ECR_REGISTRY/law-ai-web:${{ github.sha }} .
          docker push $ECR_REGISTRY/law-ai-web:${{ github.sha }}

  migrate:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      
      - name: Run migrations
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: pnpm --filter @law-ai/db migrate:latest

  deploy:
    needs: migrate
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
          
      - name: Deploy API to ECS
        run: |
          aws ecs update-service \
            --cluster law-ai-cluster \
            --service law-ai-api \
            --force-new-deployment
            
      - name: Deploy Web to ECS
        run: |
          aws ecs update-service \
            --cluster law-ai-cluster \
            --service law-ai-web \
            --force-new-deployment
```

## Secrets Management

### GitHub Secrets Required

| Secret | Purpose |
|--------|---------|
| `AWS_ACCESS_KEY_ID` | AWS deployment credentials |
| `AWS_SECRET_ACCESS_KEY` | AWS deployment credentials |
| `AWS_ACCOUNT_ID` | For ECR registry URL |
| `DATABASE_URL` | Production database connection |

### AWS Secrets Manager

Application secrets stored in AWS Secrets Manager, fetched at runtime by ECS:

| Secret | Contents |
|--------|----------|
| `law-ai/prod/api` | `OPENAI_API_KEY`, `STRIPE_SECRET_KEY`, `BETTER_AUTH_SECRET`, etc. |
| `law-ai/prod/db` | `DATABASE_URL` (also used by migrations) |
| `law-ai/prod/m365` | `MS_CLIENT_ID`, `MS_CLIENT_SECRET`, `MS_CONNECTOR_CLIENT_ID`, etc. |

ECS task definition references these secrets:

```json
{
  "secrets": [
    {
      "name": "OPENAI_API_KEY",
      "valueFrom": "arn:aws:secretsmanager:ap-southeast-2:xxx:secret:law-ai/prod/api:OPENAI_API_KEY::"
    }
  ]
}
```

## Migration Safety

### Rules for Safe Migrations

1. **Backward-compatible only** - New code must work with old schema during rollout
2. **Additive preferred** - Add columns/tables, don't remove or rename
3. **Multi-step breaking changes**:
   ```
   Deploy 1: Add new column (nullable)
   Deploy 2: Backfill data, update code to use new column
   Deploy 3: Remove old column (if needed)
   ```

### Migration Failure Handling

If migration fails:
- Deploy aborts
- No new code is deployed
- Alert sent (GitHub Actions failure notification)
- Manual intervention required

## Rollback Strategy

### Quick Rollback

```bash
# Revert to previous image
aws ecs update-service \
  --cluster law-ai-cluster \
  --service law-ai-api \
  --task-definition law-ai-api:<previous-revision>
```

### Database Rollback

Kysely supports down migrations, but **prefer forward-fixing**:
- Write a new migration to fix the issue
- Faster and safer than rolling back

## Monitoring Deploys

### Health Checks

ECS performs health checks before routing traffic:

```json
{
  "healthCheck": {
    "command": ["CMD-SHELL", "curl -f http://localhost:3001/trpc/health.check || exit 1"],
    "interval": 30,
    "timeout": 5,
    "retries": 3
  }
}
```

### Deploy Notifications

GitHub Actions can notify on deploy:

```yaml
- name: Notify Slack
  if: success()
  run: |
    curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
      -d '{"text":"Deployed to production: ${{ github.sha }}"}'
```

## Local Development

```bash
# Start local services
docker-compose up -d  # PostgreSQL + Redis

# Run migrations locally
pnpm --filter @law-ai/db migrate:latest

# Start dev servers
pnpm dev  # Runs api + web in parallel
```

## Future Improvements

| Improvement | When |
|-------------|------|
| Add staging environment | When team grows or more testing needed |
| Blue/green deployments | When zero-downtime is critical |
| Preview environments per PR | When demoing features to stakeholders |
| Automated E2E tests | Post-MVP |
