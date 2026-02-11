# Deployment Architect - Workflows Documentation

## Workflow Overview

This document outlines the operational workflows for deploying applications across different environments using Deployment Architect. Each workflow includes step-by-step procedures, decision points, rollback strategies, and best practices.

## Standard Deployment Workflows

### 1. Development Environment Deployment

**Trigger**: Automatic on push to `develop` branch

**Workflow Steps**:
```
1. Code Push
   ├─> Git webhook triggers pipeline
   └─> Extract commit metadata (author, message, SHA)

2. Build Phase
   ├─> Run unit tests
   ├─> Build Docker image
   ├─> Tag image: {service}:dev-{short-sha}
   └─> Push to development registry

3. Security Scan
   ├─> Scan for CVEs (Trivy/Snyk)
   ├─> Check for secrets (GitGuardian)
   └─> Generate SBOM

4. Deploy to Dev
   ├─> Update Kubernetes deployment
   ├─> Apply ConfigMaps/Secrets
   ├─> Run database migrations (if needed)
   └─> Wait for pods to be ready

5. Smoke Tests
   ├─> Health check endpoint
   ├─> Basic functionality tests
   └─> Log aggregation validation

6. Notification
   └─> Slack notification with deployment status
```

**Configuration Example**:
```yaml
# .deploy/dev-config.yaml
environment: development
strategy: recreate
replicas: 1
resources:
  limits:
    cpu: 500m
    memory: 512Mi
autoDeploy: true
healthCheck:
  path: /health
  initialDelay: 10s
  timeout: 5s
```

**Expected Duration**: 3-5 minutes

---

### 2. Staging Environment Deployment

**Trigger**: Manual approval after dev deployment success

**Workflow Steps**:
```
1. Manual Approval
   ├─> Reviewer checks dev environment
   ├─> Verifies test coverage metrics
   └─> Approves staging promotion

2. Pre-Deployment Validation
   ├─> Check staging cluster capacity
   ├─> Verify database migration compatibility
   ├─> Validate configuration changes
   └─> Review dependency updates

3. Build Promotion
   ├─> Re-tag image: {service}:staging-{version}
   ├─> Sign image with Cosign
   └─> Push to staging registry

4. Infrastructure Update
   ├─> Apply Terraform changes (if any)
   ├─> Wait for infrastructure stability
   └─> Update DNS records

5. Blue-Green Deployment
   ├─> Deploy to green environment
   ├─> Run integration tests on green
   ├─> Smoke test on green environment
   └─> Switch traffic from blue to green

6. Post-Deployment Testing
   ├─> Run full test suite
   ├─> Load testing (20% of production traffic)
   ├─> Security testing (DAST)
   └─> Performance benchmarking

7. Monitoring Setup
   ├─> Configure alerts
   ├─> Set up dashboards
   └─> Enable tracing

8. Notification & Documentation
   ├─> Update release notes
   ├─> Notify stakeholders
   └─> Document configuration changes
```

**Configuration Example**:
```yaml
# .deploy/staging-config.yaml
environment: staging
strategy: bluegreen
replicas: 2
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
healthCheck:
  path: /health
  initialDelay: 15s
  timeout: 10s
trafficSwitch:
  testDuration: 5m
  rollbackOnError: true
postDeploymentTests:
  - integration
  - smoke
  - load
```

**Expected Duration**: 15-20 minutes

---

### 3. Production Environment Deployment

**Trigger**: Manual approval + scheduled maintenance window

**Workflow Steps**:
```
1. Pre-Deployment Checklist
   ├─> ✓ Staging tests passed
   ├─> ✓ Security scan clean
   ├─> ✓ Release notes prepared
   ├─> ✓ Rollback plan documented
   ├─> ✓ On-call team notified
   ├─> ✓ Maintenance window scheduled
   └─> ✓ Database backup completed

2. Approval Gate
   ├─> Require 2 approvals (tech lead + product)
   ├─> Verify maintenance window
   └─> Confirm rollback readiness

3. Pre-Deployment Actions
   ├─> Scale up infrastructure (if needed)
   ├─> Create snapshot of current state
   ├─> Disable auto-scaling temporarily
   └─> Set deployment start marker

4. Canary Deployment (Progressive)
   
   Phase 1: Initial Canary (5%)
   ├─> Deploy canary pods
   ├─> Route 5% traffic to canary
   ├─> Monitor for 10 minutes
   │   ├─> Error rate < 0.1%
   │   ├─> Latency p99 < 500ms
   │   └─> No critical alerts
   └─> Decision: Continue or Rollback
   
   Phase 2: Expand Canary (25%)
   ├─> Route 25% traffic to canary
   ├─> Monitor for 10 minutes
   └─> Validate SLOs
   
   Phase 3: Half Rollout (50%)
   ├─> Route 50% traffic to canary
   ├─> Monitor for 15 minutes
   └─> Compare metrics with baseline
   
   Phase 4: Full Rollout (100%)
   ├─> Route 100% traffic to new version
   ├─> Terminate old version pods
   └─> Update load balancer

5. Post-Deployment Validation
   ├─> Run synthetic transactions
   ├─> Verify database connectivity
   ├─> Check external integrations
   ├─> Validate caching layers
   └─> Test critical user flows

6. Monitoring & Alerting
   ├─> Watch real-time metrics for 30 minutes
   ├─> Review error logs
   ├─> Monitor resource utilization
   └─> Track business metrics

7. Communication
   ├─> Update status page
   ├─> Notify customers (if needed)
   ├─> Internal announcement
   └─> Update deployment log

8. Post-Deployment Tasks
   ├─> Re-enable auto-scaling
   ├─> Clean up old artifacts
   ├─> Update documentation
   └─> Schedule retrospective
```

**Configuration Example**:
```yaml
# .deploy/production-config.yaml
environment: production
strategy: canary
replicas: 10
resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi
canary:
  steps:
    - weight: 5
      pause: 10m
    - weight: 25
      pause: 10m
    - weight: 50
      pause: 15m
    - weight: 100
  analysis:
    interval: 1m
    threshold: 5
    metrics:
      - name: error_rate
        max: 0.1
      - name: latency_p99
        max: 500
healthCheck:
  path: /health
  initialDelay: 30s
  timeout: 10s
  failureThreshold: 3
autoRollback:
  enabled: true
  conditions:
    - error_rate > 1%
    - latency_p99 > 1000ms
    - availability < 99.9%
```

**Expected Duration**: 45-60 minutes

---

## Specialized Workflows

### 4. Hotfix Deployment Workflow

**Trigger**: Critical production bug requiring immediate fix

**Fast-Track Process**:
```
1. Hotfix Creation
   ├─> Create hotfix branch from production tag
   ├─> Apply minimal fix
   ├─> Fast-track code review (1 approver)
   └─> Run critical tests only

2. Emergency Build
   ├─> Build with hotfix tag: {service}:hotfix-{issue-id}
   ├─> Quick security scan (blocking CVEs only)
   └─> Push to production registry

3. Expedited Deployment
   ├─> Skip staging (emergency override)
   ├─> Deploy to 1 pod first (validation)
   ├─> If stable after 5 min, deploy to all
   └─> Monitor closely for 30 minutes

4. Post-Hotfix Actions
   ├─> Merge hotfix to main branch
   ├─> Deploy to lower environments
   ├─> Document incident
   └─> Schedule proper fix if needed
```

**Approval Requirements**: 
- Critical: 1 tech lead approval
- Major: 1 senior engineer approval
- Communication to on-call team required

**Expected Duration**: 10-15 minutes

---

### 5. Rollback Workflow

**Trigger**: Automated alert or manual decision

**Immediate Rollback Steps**:
```
1. Rollback Decision
   ├─> Automated triggers:
   │   ├─> Error rate > 5%
   │   ├─> Availability < 99%
   │   └─> Critical alerts firing
   └─> Manual triggers:
       ├─> Customer reports
       ├─> Failed smoke tests
       └─> Business logic errors

2. Execute Rollback
   ├─> Identify last stable version
   ├─> Switch traffic to previous version (Kubernetes)
   │   └─> kubectl rollout undo deployment/{service}
   ├─> Verify pods are healthy
   └─> Update load balancer

3. Database Rollback (if needed)
   ├─> Check for schema changes
   ├─> Run down migrations
   └─> Restore from backup (last resort)

4. Validation
   ├─> Run smoke tests
   ├─> Verify error rate normalized
   ├─> Check customer-facing features
   └─> Monitor for 15 minutes

5. Post-Rollback
   ├─> Incident report
   ├─> Root cause analysis
   ├─> Communication to stakeholders
   └─> Plan fix and re-deployment
```

**Rollback Commands**:
```bash
# Kubernetes rollback
kubectl rollout undo deployment/myapp -n production

# With specific revision
kubectl rollout undo deployment/myapp --to-revision=5

# Verify rollback
kubectl rollout status deployment/myapp -n production

# AWS ECS rollback
aws ecs update-service \
  --cluster production \
  --service myapp \
  --task-definition myapp:previous-version \
  --force-new-deployment
```

**Expected Duration**: 3-5 minutes

---

### 6. Multi-Region Deployment Workflow

**Trigger**: Scheduled global rollout

**Regional Deployment Strategy**:
```
1. Region Selection Order
   ├─> Region 1: us-west-2 (canary region)
   ├─> Region 2: eu-west-1
   ├─> Region 3: ap-southeast-1
   └─> Region 4: us-east-1 (primary)

2. Per-Region Deployment
   ├─> Deploy to canary region first
   ├─> Monitor for 30 minutes
   ├─> If stable, deploy to next region
   ├─> Stagger deployments by 15 minutes
   └─> Final region: primary (most traffic)

3. Traffic Management
   ├─> Use GeoDNS for routing
   ├─> Gradual traffic shift per region
   └─> Monitor cross-region metrics

4. Failure Handling
   ├─> If region fails, halt global rollout
   ├─> Rollback failed region
   ├─> Investigate before continuing
   └─> Document region-specific issues
```

**Configuration Example**:
```yaml
# .deploy/multi-region-config.yaml
regions:
  - name: us-west-2
    role: canary
    weight: 10%
    waitTime: 30m
  - name: eu-west-1
    role: secondary
    weight: 25%
    waitTime: 15m
  - name: ap-southeast-1
    role: secondary
    weight: 25%
    waitTime: 15m
  - name: us-east-1
    role: primary
    weight: 40%
    waitTime: 20m
failurePolicy:
  stopOnRegionFailure: true
  rollbackFailedRegion: true
```

**Expected Duration**: 2-3 hours

---

## Database Migration Workflows

### 7. Safe Database Migration Workflow

**Backward-Compatible Migrations**:
```
1. Pre-Migration Phase
   ├─> Write migration scripts (up/down)
   ├─> Test migrations on dev/staging
   ├─> Create database backup
   └─> Document rollback procedure

2. Deployment Strategy
   
   Step 1: Deploy code with dual writes
   ├─> Application writes to old AND new schema
   ├─> Reads still from old schema
   └─> Deploy this version to production
   
   Step 2: Run migration
   ├─> Execute migration during low-traffic window
   ├─> Backfill data (if needed)
   └─> Verify data integrity
   
   Step 3: Switch reads to new schema
   ├─> Deploy code to read from new schema
   ├─> Monitor for errors
   └─> Keep dual writes for safety
   
   Step 4: Clean up old schema (1 week later)
   ├─> Remove dual write logic
   ├─> Drop old columns/tables
   └─> Deploy final version

3. Monitoring
   ├─> Track migration progress
   ├─> Monitor database performance
   ├─> Watch for deadlocks
   └─> Alert on replication lag
```

**Migration Script Example**:
```sql
-- V1__add_user_email_index.sql
BEGIN;

-- Add index concurrently (non-blocking)
CREATE INDEX CONCURRENTLY idx_users_email 
ON users(email);

-- Verify index created
DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_indexes 
    WHERE indexname = 'idx_users_email'
  ) THEN
    RAISE EXCEPTION 'Index creation failed';
  END IF;
END $$;

COMMIT;
```

---

## Emergency Procedures

### 8. System Outage Recovery Workflow

**Full Service Restoration**:
```
1. Incident Detection (< 1 minute)
   ├─> Automated monitoring alerts
   ├─> Customer reports
   └─> Health check failures

2. Initial Response (< 5 minutes)
   ├─> Acknowledge incident
   ├─> Page on-call engineer
   ├─> Create incident channel (#incident-xxx)
   └─> Update status page

3. Diagnosis (< 10 minutes)
   ├─> Check recent deployments
   ├─> Review error logs
   ├─> Analyze metrics spike
   └─> Identify root cause

4. Mitigation (< 15 minutes)
   ├─> Option A: Rollback deployment
   ├─> Option B: Scale up resources
   ├─> Option C: Fail over to backup region
   └─> Option D: Enable maintenance mode

5. Verification (< 5 minutes)
   ├─> Test critical user flows
   ├─> Verify error rate normalized
   ├─> Check system health
   └─> Validate data consistency

6. Recovery (< 30 minutes)
   ├─> Gradually restore traffic
   ├─> Monitor closely
   └─> Communicate status

7. Post-Incident (within 48 hours)
   ├─> Write incident report
   ├─> Conduct blameless postmortem
   ├─> Identify action items
   └─> Update runbooks
```

**SLA Targets**:
- Detection: < 1 minute
- Response: < 5 minutes
- Mitigation: < 15 minutes
- Full Recovery: < 30 minutes

---

## Automation Scripts

### Deployment Script Example

```bash
#!/bin/bash
# deploy.sh - Main deployment orchestration script

set -euo pipefail

# Configuration
SERVICE_NAME="${1:?Service name required}"
VERSION="${2:?Version required}"
ENVIRONMENT="${3:?Environment required}"
STRATEGY="${4:-rolling}"

# Functions
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"
}

check_prerequisites() {
    log "Checking prerequisites..."
    command -v kubectl >/dev/null 2>&1 || { echo "kubectl required"; exit 1; }
    command -v docker >/dev/null 2>&1 || { echo "docker required"; exit 1; }
}

build_image() {
    log "Building Docker image..."
    docker build -t "${SERVICE_NAME}:${VERSION}" .
    docker tag "${SERVICE_NAME}:${VERSION}" "registry.example.com/${SERVICE_NAME}:${VERSION}"
    docker push "registry.example.com/${SERVICE_NAME}:${VERSION}"
}

run_tests() {
    log "Running tests..."
    docker run --rm "${SERVICE_NAME}:${VERSION}" npm test
}

deploy_to_k8s() {
    log "Deploying to Kubernetes..."
    
    # Update deployment image
    kubectl set image deployment/${SERVICE_NAME} \
        ${SERVICE_NAME}=registry.example.com/${SERVICE_NAME}:${VERSION} \
        -n ${ENVIRONMENT}
    
    # Wait for rollout
    kubectl rollout status deployment/${SERVICE_NAME} \
        -n ${ENVIRONMENT} \
        --timeout=300s
}

run_smoke_tests() {
    log "Running smoke tests..."
    
    # Get service URL
    SERVICE_URL=$(kubectl get svc ${SERVICE_NAME} \
        -n ${ENVIRONMENT} \
        -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    
    # Health check
    curl -f "http://${SERVICE_URL}/health" || {
        log "Smoke test failed"
        return 1
    }
}

send_notification() {
    local status=$1
    local message="${SERVICE_NAME} v${VERSION} deployment to ${ENVIRONMENT}: ${status}"
    
    curl -X POST "${SLACK_WEBHOOK_URL}" \
        -H 'Content-Type: application/json' \
        -d "{\"text\": \"${message}\"}"
}

# Main deployment flow
main() {
    log "Starting deployment of ${SERVICE_NAME} v${VERSION} to ${ENVIRONMENT}"
    
    check_prerequisites
    build_image
    run_tests
    
    if deploy_to_k8s && run_smoke_tests; then
        log "Deployment successful"
        send_notification "SUCCESS ✓"
        exit 0
    else
        log "Deployment failed, initiating rollback"
        kubectl rollout undo deployment/${SERVICE_NAME} -n ${ENVIRONMENT}
        send_notification "FAILED ✗ (rolled back)"
        exit 1
    fi
}

main
```

### Health Check Script

```python
#!/usr/bin/env python3
"""
health_check.py - Deployment health validation
"""

import requests
import time
import sys

def check_health(url, retries=10, delay=5):
    """Check service health with retries"""
    for i in range(retries):
        try:
            response = requests.get(f"{url}/health", timeout=5)
            if response.status_code == 200:
                data = response.json()
                print(f"✓ Health check passed: {data}")
                return True
            else:
                print(f"✗ Health check failed: HTTP {response.status_code}")
        except requests.exceptions.RequestException as e:
            print(f"✗ Connection error: {e}")
        
        if i < retries - 1:
            print(f"Retrying in {delay} seconds... ({i+1}/{retries})")
            time.sleep(delay)
    
    return False

def check_metrics(url):
    """Validate deployment metrics"""
    try:
        response = requests.get(f"{url}/metrics")
        if 'deployment_version' in response.text:
            print("✓ Metrics endpoint accessible")
            return True
    except:
        pass
    
    print("✗ Metrics endpoint failed")
    return False

def main():
    if len(sys.argv) < 2:
        print("Usage: health_check.py <service-url>")
        sys.exit(1)
    
    service_url = sys.argv[1]
    
    print(f"Checking health of {service_url}...")
    
    health_ok = check_health(service_url)
    metrics_ok = check_metrics(service_url)
    
    if health_ok and metrics_ok:
        print("\n✓ All checks passed")
        sys.exit(0)
    else:
        print("\n✗ Health checks failed")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## Best Practices

### Deployment Checklist

**Pre-Deployment**:
- [ ] Code review completed
- [ ] All tests passing
- [ ] Security scan clean
- [ ] Database migrations tested
- [ ] Configuration validated
- [ ] Rollback plan documented
- [ ] On-call team notified
- [ ] Monitoring configured

**During Deployment**:
- [ ] Watch deployment logs
- [ ] Monitor error rates
- [ ] Track resource utilization
- [ ] Verify health checks
- [ ] Test critical flows
- [ ] Keep communication channel open

**Post-Deployment**:
- [ ] Verify metrics baseline
- [ ] Run integration tests
- [ ] Update documentation
- [ ] Notify stakeholders
- [ ] Monitor for 30+ minutes
- [ ] Clean up old artifacts

### Troubleshooting Guide

**Common Issues**:

1. **Pods not starting**:
   ```bash
   kubectl describe pod <pod-name> -n <namespace>
   kubectl logs <pod-name> -n <namespace>
   ```

2. **Image pull errors**:
   - Verify registry credentials
   - Check image tag exists
   - Review network policies

3. **Database migration failures**:
   - Check database connectivity
   - Verify migration scripts
   - Review transaction logs

4. **Health check failures**:
   - Increase initialDelaySeconds
   - Check application startup time
   - Verify health endpoint logic

---

## Metrics and KPIs

**Track these metrics**:
- Deployment frequency (per day/week)
- Lead time for changes (commit to deploy)
- Mean time to recovery (MTTR)
- Change failure rate (%)
- Deployment success rate (%)
- Rollback frequency (%)

**Target SLOs**:
- Deployment success rate: > 95%
- MTTR: < 30 minutes
- Change failure rate: < 5%
- Zero-downtime deployments: 100%
