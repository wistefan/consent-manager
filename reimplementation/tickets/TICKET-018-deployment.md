# TICKET-018: Deployment Configuration

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-018 |
| **Title** | Deployment Configuration |
| **Priority** | Medium |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-001 |
| **Related User Stories** | (infrastructure — no direct user stories) |

## Description

Create the deployment infrastructure for the Consent Manager including Docker configuration, Docker Compose for local development, Kubernetes Helm chart templates, Terraform configuration, and environment configuration management.

## Technical Details

### Docker Configuration

#### Application Dockerfile (`docker/app/Dockerfile`)

Multi-stage build:

**Stage 1: Build**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm run build
```

**Stage 2: Production**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
COPY --from=builder /app/docs ./docs
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

#### MongoDB Dockerfile (`docker/mongodb/Dockerfile`)

MongoDB with replica set initialization for local development:

```dockerfile
FROM mongo:7
COPY init-replica.sh /docker-entrypoint-initdb.d/
```

### Docker Compose (`docker-compose.yml`)

```yaml
services:
  app:
    build:
      context: .
      dockerfile: docker/app/Dockerfile
    ports:
      - "3000:3000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/consent-manager?replicaSet=rs0
      # ... other env vars
    depends_on:
      - mongodb

  mongodb:
    build:
      context: docker/mongodb
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    command: ["--replSet", "rs0"]

volumes:
  mongodb_data:
```

### Kubernetes Helm Chart

#### Chart Metadata (`Chart.yaml`)

```yaml
apiVersion: v2
name: consent-manager
description: Prometheus-X Consent Manager
version: 1.2.0
appVersion: "1.2.0"
```

#### Helm Templates (`templates/`)

| Template | Purpose |
|----------|---------|
| `deployment.yaml` | Kubernetes Deployment with configurable replicas, resources, probes |
| `service.yaml` | ClusterIP Service exposing port 3000 |
| `secret.yaml` | Kubernetes Secret for sensitive env vars (JWT keys, API keys, DB URI) |
| `configmap.yaml` | ConfigMap for non-sensitive configuration |
| `ingress.yaml` | Optional Ingress for external access |
| `hpa.yaml` | Optional Horizontal Pod Autoscaler |

#### Values (`values.yaml`)

```yaml
replicaCount: 1

image:
  repository: consent-manager
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 3000

ingress:
  enabled: false
  # ...

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

mongodb:
  uri: ""  # Set via secret

env:
  API_PREFIX: "/v1"
  NODE_ENV: "production"
  # ...
```

#### Health Probes

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 15
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Terraform Configuration

**Directory:** `terraform/`

| File | Purpose |
|------|---------|
| `main.tf` | Provider configuration and resource definitions |
| `variables.tf` | Input variables for deployment customization |
| `outputs.tf` | Output values (service URLs, connection strings) |

### Environment Configuration

#### `.env.sample`

Document all environment variables with descriptions and example values:

```bash
# Server
PORT=3000
NODE_ENV=development
API_PREFIX=/v1

# MongoDB
MONGODB_URI=mongodb://localhost:27017/consent-manager?replicaSet=rs0

# Authentication
JWT_SECRET_KEY=your-jwt-secret-key
OAUTH_SECRET_KEY=your-oauth-secret-key
SESSION_SECRET=your-session-secret
X_CATALOG_CONSENT_KEY=your-catalog-key

# Service URLs
CONSENT_MANAGER_URL=http://localhost:3000
CONTRACT_SERVICE_URL=http://localhost:3001
PDI_FRONTEND_URL=http://localhost:3002

# Email (Mandrill)
MANDRILL_API_KEY=

# Email (Nodemailer)
NODEMAILER_HOST=
NODEMAILER_PORT=
NODEMAILER_USER=
NODEMAILER_PASS=

# Feature Flags
ENABLE_OAUTH=false
```

### Files to Create

```
docker/
  app/Dockerfile
  mongodb/
    Dockerfile
    init-replica.sh
docker-compose.yml
templates/
  deployment.yaml
  service.yaml
  secret.yaml
  configmap.yaml
  ingress.yaml
Chart.yaml
values.yaml
terraform/
  main.tf
  variables.tf
  outputs.tf
.env.sample
.dockerignore
```

## Acceptance Criteria

1. [ ] `docker build` creates a production image from the multi-stage Dockerfile
2. [ ] `docker-compose up` starts the app and MongoDB with replica set
3. [ ] Application connects to MongoDB and passes health check within Docker Compose
4. [ ] Helm chart installs successfully with `helm install`
5. [ ] Helm chart configures liveness and readiness probes using `/health`
6. [ ] Secrets are managed via Kubernetes Secret resources (not plain ConfigMaps)
7. [ ] `.env.sample` documents every environment variable used by the application
8. [ ] `.dockerignore` excludes `node_modules`, `.env`, key files, and test files
9. [ ] Terraform files define the basic infrastructure resources

## Known Issues to Address

1. **MongoDB replica set requirement**: Document that MongoDB must be running as a replica set for Mongoose transactions and change streams. Provide the `init-replica.sh` script for local development.
2. **Key file distribution**: Document how RSA and AES keys should be provisioned in production (Kubernetes secrets, mounted volumes, etc.).
3. **Environment variable sprawl**: Consider grouping related variables and providing sensible defaults to reduce required configuration.
