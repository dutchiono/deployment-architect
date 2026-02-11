# Deployment Architect - System Architecture

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                  Deployment Architect System                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │   CI/CD      │      │  Container   │      │ Infrastructure│ │
│  │  Pipeline    │──────│ Orchestration│──────│   as Code    │  │
│  └──────────────┘      └──────────────┘      └──────────────┘  │
│         │                      │                      │          │
│         │                      │                      │          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │            Deployment Strategy Engine                     │  │
│  │  • Blue-Green / Canary / Rolling Update Selection        │  │
│  │  • Progressive Delivery & Traffic Shifting               │  │
│  │  • Rollback Automation & Health Monitoring               │  │
│  │  • Multi-Environment Orchestration                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│         │                      │                      │          │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │   GitOps     │      │   Security   │      │  Observability│ │
│  │  Controller  │──────│   Scanner    │──────│   & Metrics  │  │
│  └──────────────┘      └──────────────┘      └──────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │
        ┌─────────────────────┴─────────────────────┐
        │                                             │
┌───────▼────────┐                          ┌────────▼───────┐
│   Target       │                          │   Monitoring   │
│   Platforms    │                          │   & Alerts     │
├────────────────┤                          ├────────────────┤
│ • Kubernetes   │                          │ • Prometheus   │
│ • Docker Swarm │                          │ • Grafana      │
│ • ECS/Fargate  │                          │ • Datadog      │
│ • Cloud Run    │                          │ • PagerDuty    │
│ • Lambda       │                          │ • Slack        │
└────────────────┘                          └────────────────┘
```

## Core Components

### 1. CI/CD Pipeline Engine
**Purpose:** Automate build, test, scan, and deployment workflows

**Pipeline Stages:**

```yaml
# .gitlab-ci.yml example
stages:
  - build
  - test
  - scan
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  KUBE_NAMESPACE: production

build:
  stage: build
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  only:
    - main

test:
  stage: test
  script:
    - docker run $DOCKER_IMAGE npm test
    - docker run $DOCKER_IMAGE npm run lint
  coverage: '/Coverage: \d+\.\d+%/'

security-scan:
  stage: scan
  script:
    - trivy image --severity HIGH,CRITICAL $DOCKER_IMAGE
    - snyk container test $DOCKER_IMAGE

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE -n $KUBE_NAMESPACE
    - kubectl rollout status deployment/myapp -n $KUBE_NAMESPACE
  environment:
    name: production
    url: https://app.example.com
```

**Key Features:**
- Parallel job execution for speed
- Artifact caching (Docker layers, dependencies)
- Conditional execution (branch, tag, manual)
- Secret injection (Vault, AWS Secrets Manager)
- Failure notifications (Slack, email, PagerDuty)

### 2. Container Orchestration
**Purpose:** Manage containerized applications at scale

**Kubernetes Architecture:**

```
┌─────────────────────────────────────────────┐
│              Kubernetes Cluster              │
├─────────────────────────────────────────────┤
│                                              │
│  ┌─────────────────────────────────────┐   │
│  │         Control Plane               │   │
│  │  • API Server                       │   │
│  │  • etcd (state store)               │   │
│  │  • Scheduler                        │   │
│  │  • Controller Manager               │   │
│  └─────────────────────────────────────┘   │
│                    │                         │
│  ┌─────────────────┴───────────────────┐   │
│  │         Worker Nodes                │   │
│  │  ┌──────────┐  ┌──────────┐        │   │
│  │  │  Pod 1   │  │  Pod 2   │        │   │
│  │  │ ┌──────┐ │  │ ┌──────┐ │        │   │
│  │  │ │ App  │ │  │ │ App  │ │        │   │
│  │  │ └──────┘ │  │ └──────┘ │        │   │
│  │  └──────────┘  └──────────┘        │   │
│  │  • kubelet                          │   │
│  │  • kube-proxy                       │   │
│  │  • container runtime (containerd)   │   │
│  └─────────────────────────────────────┘   │
│                                              │
└─────────────────────────────────────────────┘
```

**Kubernetes Resources:**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
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
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1.2.3
    spec:
      containers:
      - name: myapp
        image: myapp:v1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 3. Deployment Strategy Engine
**Purpose:** Execute zero-downtime deployments with automated rollback

**Strategy Comparison:**

| Strategy | Downtime | Risk | Rollback Speed | Cost | Use Case |
|----------|----------|------|----------------|------|----------|
| Blue-Green | Zero | Low | Instant | High (2x resources) | Critical apps |
| Canary | Zero | Low | Fast | Medium | Gradual rollout |
| Rolling | Near-zero | Medium | Medium | Low | Standard updates |
| Recreate | Minutes | High | N/A | Low | Dev/test only |

**Blue-Green Deployment:**

```yaml
# Blue environment (current production)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue
  ports:
  - port: 80

---
# Green environment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0.0

# After validation, switch traffic:
# kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'
```

**Canary Deployment with Istio:**

```yaml
# Virtual Service for traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: myapp
        subset: canary
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 90
    - destination:
        host: myapp
        subset: canary
      weight: 10  # 10% canary traffic

---
# Destination Rule for subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: stable
    labels:
      version: v1
  - name: canary
    labels:
      version: v2
```

**Progressive Delivery with Flagger:**

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
    webhooks:
    - name: load-test
      url: http://flagger-loadtester/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://myapp/"
```

### 4. Infrastructure as Code (IaC)
**Purpose:** Define infrastructure declaratively with version control

**Terraform Architecture:**

```hcl
# main.tf - EKS cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "production-cluster"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    general = {
      desired_size = 3
      min_size     = 2
      max_size     = 10

      instance_types = ["t3.large"]
      capacity_type  = "ON_DEMAND"

      labels = {
        Environment = "production"
        Role        = "general"
      }

      taints = []
    }

    spot = {
      desired_size = 2
      min_size     = 0
      max_size     = 5

      instance_types = ["t3.large", "t3.xlarge"]
      capacity_type  = "SPOT"

      labels = {
        Environment = "production"
        Role        = "batch"
      }

      taints = [{
        key    = "spot"
        value  = "true"
        effect = "NoSchedule"
      }]
    }
  }

  tags = {
    Environment = "production"
    Terraform   = "true"
  }
}

# rds.tf - Database
resource "aws_db_instance" "main" {
  identifier        = "myapp-prod"
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.r6g.large"
  allocated_storage = 100

  db_name  = "myapp"
  username = "admin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  tags = {
    Environment = "production"
  }
}

# outputs.tf
output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "database_endpoint" {
  value = aws_db_instance.main.endpoint
}
```

**GitOps with ArgoCD:**

```yaml
# Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-k8s
    targetRevision: main
    path: overlays/production
    helm:
      values: |
        replicaCount: 3
        image:
          tag: v1.2.3
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### 5. Security Scanner
**Purpose:** Identify vulnerabilities before deployment

**Security Scanning Pipeline:**

```yaml
# Container scanning with Trivy
container-scan:
  stage: scan
  script:
    - trivy image --exit-code 1 --severity CRITICAL $DOCKER_IMAGE
    - trivy image --exit-code 0 --severity HIGH,MEDIUM $DOCKER_IMAGE
  allow_failure:
    exit_codes: [0, 1]

# SAST (Static Application Security Testing)
sast:
  stage: scan
  script:
    - semgrep --config=auto --json -o sast-report.json .
  artifacts:
    reports:
      sast: sast-report.json

# Dependency scanning
dependency-scan:
  stage: scan
  script:
    - npm audit --audit-level=moderate
    - snyk test --severity-threshold=high

# IaC scanning
iac-scan:
  stage: scan
  script:
    - tfsec --format json terraform/
    - checkov -d terraform/ --framework terraform
```

### 6. Observability & Monitoring
**Purpose:** Track deployment health and performance

**Monitoring Stack:**

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics

---
# Grafana Dashboard ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "title": "MyApp Deployment Metrics",
      "panels": [
        {
          "title": "Request Rate",
          "targets": [
            {
              "expr": "rate(http_requests_total[5m])"
            }
          ]
        },
        {
          "title": "Error Rate",
          "targets": [
            {
              "expr": "rate(http_requests_total{status=~\"5..\"}[5m])"
            }
          ]
        },
        {
          "title": "Pod Restarts",
          "targets": [
            {
              "expr": "kube_pod_container_status_restarts_total"
            }
          ]
        }
      ]
    }
```

## Deployment Patterns

### Pattern 1: Multi-Region Active-Active

```
┌─────────────────────────────────────────────┐
│           Global Load Balancer              │
│         (AWS Global Accelerator)            │
└─────────────┬───────────────┬───────────────┘
              │               │
    ┌─────────▼──────┐  ┌────▼──────────┐
    │  us-east-1     │  │  eu-west-1    │
    │  EKS Cluster   │  │  EKS Cluster  │
    │  • App Pods    │  │  • App Pods   │
    │  • RDS Primary │  │  • RDS Replica│
    └────────────────┘  └───────────────┘
```

### Pattern 2: Multi-Environment Pipeline

```
Commit → Build
           ↓
       Container Registry
           ↓
    ┌──────┴──────┐
    ↓             ↓
  Dev Env    Staging Env
    ↓             ↓
 Auto Deploy  Auto Deploy
               ↓
         Integration Tests
               ↓
         Manual Approval
               ↓
         Production Env
               ↓
         Canary Deploy (10%)
               ↓
         Monitor (1 hour)
               ↓
    Full Rollout (100%)
```

## Technology Stack

**CI/CD Tools:**
- GitHub Actions, GitLab CI, Jenkins, CircleCI
- ArgoCD, Flux (GitOps)
- Spinnaker (multi-cloud CD)

**Container Platforms:**
- Kubernetes (EKS, GKE, AKS)
- Docker Swarm
- AWS ECS/Fargate
- Google Cloud Run

**IaC Tools:**
- Terraform
- AWS CloudFormation
- Pulumi
- Ansible

**Security:**
- Trivy, Snyk (vulnerability scanning)
- Falco (runtime security)
- OPA/Gatekeeper (policy enforcement)

**Monitoring:**
- Prometheus + Grafana
- Datadog, New Relic
- ELK Stack (logs)
- Jaeger (tracing)

## Best Practices

1. **Immutable Infrastructure**: Never modify running containers; deploy new versions
2. **Environment Parity**: Keep dev, staging, production as similar as possible
3. **Automated Testing**: Run unit, integration, e2e tests before deploy
4. **Gradual Rollout**: Use canary or blue-green for critical services
5. **Rollback Plan**: Always have automated rollback capability
6. **Security First**: Scan images, enforce policies, rotate secrets
7. **Observability**: Instrument code, collect metrics, set up alerts
8. **Version Everything**: Git for code, IaC, configs, documentation

## References

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [The Twelve-Factor App](https://12factor.net/)
- [GitOps Principles](https://www.gitops.tech/)
- [Google SRE Book](https://sre.google/books/)
