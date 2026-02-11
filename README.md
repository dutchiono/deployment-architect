# Deployment Architect

> Deployment pipeline and infrastructure automation system for containerized applications

## Overview

Deployment Architect is a comprehensive AI agent specialized in designing, implementing, and managing CI/CD pipelines, container orchestration, infrastructure automation, and deployment strategies for modern cloud-native applications. It handles Docker containerization, Kubernetes orchestration, GitOps workflows, infrastructure as code, and progressive delivery patterns.

## Core Capabilities

### CI/CD Pipeline Design
- Multi-stage pipeline architecture (build, test, scan, deploy)
- Parallel job execution and dependency management
- Artifact management and versioning
- Secret management and credential rotation
- Pipeline optimization and caching strategies

### Container Orchestration
- Kubernetes cluster design and management
- Helm chart development and templating
- Resource management (requests, limits, autoscaling)
- Service mesh integration (Istio, Linkerd)
- Multi-tenancy and namespace isolation

### Deployment Strategies
- Blue-Green deployments (zero-downtime switches)
- Canary deployments (progressive rollout with metrics)
- Rolling updates (controlled pod replacement)
- Feature flags and A/B testing integration
- Rollback automation and disaster recovery

### Infrastructure as Code
- Terraform modules for cloud resources
- CloudFormation / ARM templates
- Ansible playbooks for configuration management
- GitOps with ArgoCD / Flux
- Policy as Code with OPA / Sentinel

### Monitoring & Observability
- Deployment health checks and readiness probes
- Metrics collection (Prometheus, Datadog)
- Log aggregation (ELK, Loki)
- Distributed tracing (Jaeger, Zipkin)
- Alert configuration and incident management

## Technology Stack

**Container Platforms:**
- Docker (BuildKit, multi-stage builds)
- Kubernetes (1.28+, EKS, GKE, AKS)
- Helm 3 (chart management, templating)
- Kustomize (declarative configuration)

**CI/CD Tools:**
- GitHub Actions (workflows, reusable actions)
- GitLab CI/CD (pipelines, runners)
- Jenkins (pipelines, shared libraries)
- CircleCI (orbs, workflows)
- Argo Workflows (Kubernetes-native CI/CD)

**GitOps:**
- ArgoCD (declarative GitOps)
- Flux CD (automated deployments)
- Tekton (cloud-native CI/CD)

**Infrastructure as Code:**
- Terraform (0.14+, modules, workspaces)
- Pulumi (TypeScript, Python, Go)
- AWS CDK (CloudFormation abstraction)
- Crossplane (Kubernetes-native IaC)

**Cloud Providers:**
- AWS (EKS, ECR, ECS, CodePipeline)
- Google Cloud (GKE, Artifact Registry)
- Azure (AKS, Container Registry)
- DigitalOcean (Kubernetes, Container Registry)

## Quick Start

### Docker Multi-Stage Build
```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
USER nodejs
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
      - name: api
        image: registry.example.com/api-service:v1.2.3
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 15
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
```

### GitHub Actions Pipeline
```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: docker/build-push-action@v5
      with:
        push: true
        tags: ghcr.io/org/app:${{ github.sha }}
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/api-service api=ghcr.io/org/app:${{ github.sha }}
        kubectl rollout status deployment/api-service
```

## Key Features

### 1. Canary Deployments
Progressive rollout with automated metrics validation

### 2. GitOps
Declarative infrastructure managed through Git

### 3. Multi-Cloud
Deploy to AWS, GCP, Azure, or on-premises

### 4. Secrets Management
Sealed secrets, Vault integration, encrypted configs

## Best Practices

1. **Immutable Infrastructure**: Rebuild containers instead of modifying
2. **Version Everything**: Tag images, Helm charts, Terraform modules
3. **Health Checks**: Always define liveness and readiness probes
4. **Resource Limits**: Set appropriate requests and limits
5. **GitOps**: Single source of truth in Git
6. **Multi-Environment**: Separate configs for dev/staging/prod

## Documentation

- [ARCHITECTURE.md](./ARCHITECTURE.md) - System architecture and pipeline design
- [INTEGRATION.md](./INTEGRATION.md) - Tool integrations and cloud provider setup
- [WORKFLOWS.md](./WORKFLOWS.md) - Deployment workflows and operational procedures

## License

Copyright © 2026. All rights reserved.
