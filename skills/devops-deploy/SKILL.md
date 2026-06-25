---
name: devops-deploy
description: >
  Build production-grade deployment infrastructure. Trigger for: Dockerizing
  services, Docker Compose for local/multi-service, GitHub Actions CI/CD pipelines,
  rollback-safe AWS deployments (EC2, ECS, ECR), Nginx config, environment config
  management, zero-downtime deploys, Jenkins pipelines, Kubernetes basics, health
  checks, secrets management, and observability (Prometheus + Grafana). Also
  trigger for "write a Dockerfile", "CI/CD pipeline", "deploy to AWS", "set up
  GitHub Actions", "zero-downtime deploy", "rollback strategy", "multi-stage
  Docker build", "Nginx reverse proxy", and "how do I deploy this".
---

# DevOps/Deploy Skill — Ship With Confidence

You build deployment pipelines that are **fast, rollback-safe, and observable.**
The goal: a broken deploy is caught before it reaches users, and rolling back is
a one-command operation.

## Dockerfile — production-grade multi-stage

**Go service:**
```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download                         # cached layer — only re-runs when go.mod changes
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server ./cmd/server

FROM gcr.io/distroless/static-debian12      # zero shell, no package manager, minimal attack surface
WORKDIR /app
COPY --from=builder /app/server .
USER nonroot:nonroot                        # never run as root
EXPOSE 8080
ENTRYPOINT ["/app/server"]
```

**Node.js / Next.js:**
```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile   # exact lockfile install

FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

Dockerfile rules:
- Always multi-stage — production image has NO build tools, NO dev deps, NO shell (distroless).
- Order layers: install deps → copy source → build. Cache deps separately.
- `.dockerignore`: `node_modules`, `.git`, `.env*`, `dist`, `*.test.*`.
- Never run as root. Use `USER nonroot` / create a non-root user.
- Never `COPY . .` before `go mod download` / `pnpm install` — kills the cache layer.

## Docker Compose (local dev + service orchestration)

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: postgres://dev:dev@postgres:5432/appdb
      REDIS_URL: redis://redis:6379
    env_file: [".env.local"]
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_started }
    restart: unless-stopped

  postgres:
    image: pgvector/pgvector:pg16    # includes the vector extension
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: appdb
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev -d appdb"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    volumes: ["redisdata:/data"]

volumes: { pgdata: {}, redisdata: {} }
```

## GitHub Actions CI/CD — the production pipeline

```yaml
# .github/workflows/deploy.yml
name: CI / Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  IMAGE: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/myapp

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_USER: test, POSTGRES_PASSWORD: test, POSTGRES_DB: test }
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 5s --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "1.23" }
      - run: go test -race -count=1 ./...
        env: { TEST_DATABASE_URL: postgres://test:test@localhost:5432/test }
      - run: go vet ./...

  deploy:
    if: github.ref == 'refs/heads/main'     # only deploy on main, not PRs
    needs: [test]                            # tests must pass first
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:            ${{ secrets.AWS_REGION }}

      - name: Build & push image
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin $IMAGE
          docker build -t $IMAGE:${{ github.sha }} -t $IMAGE:latest .
          docker push $IMAGE:${{ github.sha }}
          docker push $IMAGE:latest

      - name: Deploy to EC2 (rollback-safe)
        run: |
          ssh -o StrictHostKeyChecking=no ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} << EOF
            # Pull new image tag (not :latest — pinned to commit SHA)
            docker pull $IMAGE:${{ github.sha }}
            # Blue/green swap: start new, stop old (zero downtime with health check)
            docker stop app-new 2>/dev/null || true
            docker run -d --name app-new --health-cmd "curl -f http://localhost:8080/health" \
              --health-interval=5s --health-retries=6 \
              -p 8081:8080 $IMAGE:${{ github.sha }}
            sleep 10 && docker inspect --format="{{.State.Health.Status}}" app-new | grep -q healthy
            # Health check passed — swap ports via Nginx reload
            docker stop app && docker rename app app-old
            docker rename app-new app && docker rm app-old
          EOF
```

Rollback strategy: pin deploys to `${{ github.sha }}` tag (never deploy `:latest`
in prod). Roll back = `docker pull $IMAGE:<prev-sha> && docker stop app && docker
run ...`. Old image is in ECR, one command away.

## Nginx reverse proxy + SSL

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;    # redirect HTTP → HTTPS
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header Content-Security-Policy "default-src 'self'";

    client_max_body_size 20m;

    location / {
        proxy_pass         http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;       # WebSocket support
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 300s;                        # for streaming/long AI calls
    }

    location /health { access_log off; proxy_pass http://localhost:8080/health; }
}
```

## Secrets management

```
NEVER: hardcoded in code, in docker-compose.yml values, committed to git.
ALWAYS: environment variables injected at runtime.

Local dev:  .env.local  (in .gitignore)
CI secrets: GitHub Actions Secrets → ${{ secrets.NAME }}
Production:
  Simple:    EC2 user-data / .env file owned root:root 600, loaded by systemd unit
  Better:    AWS Parameter Store (free) or Secrets Manager → injected by ECS task def
  Best:      AWS Secrets Manager + IAM role for the instance (no static credentials)

Rotate secrets: never reuse across envs (prod/staging/dev have separate DB creds).
Audit: run `git log -S 'password\|secret\|key' -- '*.env'` periodically.
```

## Health checks (required for zero-downtime)

```go
// Every service must expose /health
mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
    // Ping critical deps; return 503 if any fail
    if err := db.Ping(r.Context()); err != nil {
        writeJSON(w, http.StatusServiceUnavailable, map[string]string{"db": "down"})
        return
    }
    writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
})
```
The load balancer / container orchestrator calls `/health` every 5s. Failing health
check = removed from rotation automatically. Healthy = traffic flows. This is what
makes zero-downtime deploys safe.

## Observability minimum (Prometheus + structured logs)

```go
// Expose Prometheus metrics from Go
import "github.com/prometheus/client_golang/prometheus/promhttp"
mux.Handle("/metrics", promhttp.Handler())

// Key metrics to expose:
httpRequestsTotal   // counter: method, path, status
httpDuration        // histogram: p50/p95/p99 latency
dbQueryDuration     // histogram: query name, duration
activeConnections   // gauge: current open connections
```

```go
// Structured log on every request (slog → JSON → your log aggregator)
slog.Info("request", "method", r.Method, "path", r.URL.Path,
    "status", status, "duration_ms", ms, "request_id", reqID)
```

Grafana RED dashboard alert thresholds:
- Error rate > 1% → page on-call.
- P99 latency > 1s → page on-call.
- Healthy instance count drops below minimum → page immediately.

## Anti-patterns

```
❌ Deploying :latest tag → can't pin a rollback target
❌ Running as root in container → privilege escalation risk
❌ Secrets in docker-compose.yml or environment stanza (visible in docker inspect)
❌ No health check → deploy looks successful but app is broken
❌ Deploy before tests pass → use needs: [test] gate
❌ One environment (no staging) → always: dev → staging → prod
❌ SSH keys in the repo → use GitHub Secrets and IAM roles
❌ Fat images (build tools in prod) → always multi-stage
❌ No resource limits in production → one OOM kills the host
```

## Response format

1. Give the specific file (Dockerfile / workflow YAML / nginx conf) — full, not partial.
2. Call out the rollback story: how do you undo this deploy in under 2 minutes.
3. Name the one security/reliability consideration and how the config handles it.
