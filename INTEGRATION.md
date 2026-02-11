# Deployment Architect - Integration Specifications

## Integration Overview

Deployment Architect serves as the central orchestrator for software delivery, integrating with source control, build systems, infrastructure providers, monitoring platforms, and security tools. It acts as the bridge between development and operations, automating the path from code commit to production deployment.

## Integration Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Deployment Architect                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │  Pipeline  │  │   Build    │  │   Deploy   │            │
│  │   Engine   │─▶│   System   │─▶│ Controller │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└───────┬────────────────┬────────────────┬───────────────────┘
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Source       │  │ Infrastructure│  │ Monitoring   │
│ Control      │  │ Providers    │  │ & Alerting   │
│ • GitHub     │  │ • AWS        │  │ • Prometheus │
│ • GitLab     │  │ • GCP        │  │ • Datadog    │
│ • Bitbucket  │  │ • Azure      │  │ • New Relic  │
└──────────────┘  └──────────────┘  └──────────────┘
```

## Source Control Integration

### GitHub Integration

**Authentication**: Personal Access Token (PAT), GitHub App, OAuth

**Integration Points**:
```yaml
# GitHub Actions Workflow
name: Deploy Pipeline
on:
  push:
    branches: [main, staging]
  pull_request:
    types: [opened, synchronize]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Trigger Deployment
        uses: deployment-architect/deploy-action@v1
        with:
          environment: ${{ github.ref_name }}
          config: .deploy/config.yaml
          secrets: ${{ secrets.DEPLOY_SECRETS }}
```

**Webhook Events**:
- `push`: Trigger deployment on branch push
- `pull_request`: Deploy preview environments
- `release`: Trigger production deployment
- `deployment_status`: Update deployment state

**API Integration**:
```python
from github import Github

def create_deployment(repo_name, ref, environment):
    g = Github(token)
    repo = g.get_repo(repo_name)
    
    deployment = repo.create_deployment(
        ref=ref,
        environment=environment,
        auto_merge=False,
        required_contexts=[]
    )
    
    return deployment.id
```

### GitLab Integration

**Authentication**: Personal Access Token, Deploy Token, OAuth

**CI/CD Configuration**:
```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

deploy:production:
  stage: deploy
  script:
    - deployment-architect deploy
        --environment production
        --config .deploy/config.yaml
  only:
    - main
  when: manual
  environment:
    name: production
    url: https://app.example.com
```

**API Integration**:
```python
import gitlab

def trigger_deployment(project_id, ref, environment):
    gl = gitlab.Gitlab(url, private_token=token)
    project = gl.projects.get(project_id)
    
    pipeline = project.pipelines.create({
        'ref': ref,
        'variables': [
            {'key': 'ENVIRONMENT', 'value': environment}
        ]
    })
    
    return pipeline.id
```

## Infrastructure Provider Integration

### AWS Integration

**Authentication**: IAM Role, Access Keys, SSO

**Service Integrations**:

**1. ECS/Fargate Deployment**:
```python
import boto3

def deploy_to_ecs(cluster, service, image):
    ecs = boto3.client('ecs')
    
    # Update task definition
    response = ecs.register_task_definition(
        family=service,
        containerDefinitions=[{
            'name': service,
            'image': image,
            'memory': 512,
            'cpu': 256,
            'essential': True,
            'portMappings': [{
                'containerPort': 8080,
                'protocol': 'tcp'
            }]
        }],
        requiresCompatibilities=['FARGATE'],
        networkMode='awsvpc',
        cpu='256',
        memory='512'
    )
    
    # Update service
    ecs.update_service(
        cluster=cluster,
        service=service,
        taskDefinition=response['taskDefinition']['taskDefinitionArn'],
        forceNewDeployment=True
    )
```

**2. Lambda Deployment**:
```python
def deploy_lambda(function_name, s3_bucket, s3_key):
    lambda_client = boto3.client('lambda')
    
    response = lambda_client.update_function_code(
        FunctionName=function_name,
        S3Bucket=s3_bucket,
        S3Key=s3_key,
        Publish=True
    )
    
    # Update alias to new version
    lambda_client.update_alias(
        FunctionName=function_name,
        Name='production',
        FunctionVersion=response['Version'],
        RoutingConfig={
            'AdditionalVersionWeights': {
                response['Version']: 0.1  # 10% canary
            }
        }
    )
```

**3. CloudFormation Stack Updates**:
```python
def update_stack(stack_name, template_url, parameters):
    cfn = boto3.client('cloudformation')
    
    cfn.update_stack(
        StackName=stack_name,
        TemplateURL=template_url,
        Parameters=parameters,
        Capabilities=['CAPABILITY_IAM'],
        Tags=[
            {'Key': 'ManagedBy', 'Value': 'DeploymentArchitect'},
            {'Key': 'Environment', 'Value': 'production'}
        ]
    )
    
    # Wait for stack update
    waiter = cfn.get_waiter('stack_update_complete')
    waiter.wait(StackName=stack_name)
```

### Google Cloud Platform Integration

**Authentication**: Service Account Key, Application Default Credentials

**Service Integrations**:

**1. GKE Deployment**:
```python
from google.cloud import container_v1

def deploy_to_gke(project_id, zone, cluster_id, deployment_yaml):
    client = container_v1.ClusterManagerClient()
    cluster_name = f'projects/{project_id}/locations/{zone}/clusters/{cluster_id}'
    
    # Apply Kubernetes manifests
    import subprocess
    subprocess.run([
        'gcloud', 'container', 'clusters', 'get-credentials',
        cluster_id, '--zone', zone, '--project', project_id
    ])
    
    subprocess.run(['kubectl', 'apply', '-f', deployment_yaml])
```

**2. Cloud Run Deployment**:
```python
from google.cloud import run_v2

def deploy_cloud_run(project_id, region, service_name, image):
    client = run_v2.ServicesClient()
    
    service = run_v2.Service(
        template=run_v2.RevisionTemplate(
            containers=[run_v2.Container(
                image=image,
                resources=run_v2.ResourceRequirements(
                    limits={'cpu': '1', 'memory': '512Mi'}
                )
            )],
            scaling=run_v2.RevisionScaling(
                min_instance_count=1,
                max_instance_count=10
            )
        ),
        traffic=[run_v2.TrafficTarget(
            type_=run_v2.TrafficTargetAllocationType.TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST,
            percent=100
        )]
    )
    
    parent = f'projects/{project_id}/locations/{region}'
    operation = client.create_service(
        parent=parent,
        service=service,
        service_id=service_name
    )
    
    return operation.result()
```

### Kubernetes Integration

**Authentication**: kubeconfig, Service Account Token

**Deployment Strategies**:

**1. Rolling Update**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    deployment-architect.io/strategy: rolling
    deployment-architect.io/health-check: /health
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0.0
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

**2. Canary Deployment with Flagger**:
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 8080
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
```

**Python Client Integration**:
```python
from kubernetes import client, config

def deploy_to_k8s(namespace, deployment_name, image):
    config.load_kube_config()
    apps_v1 = client.AppsV1Api()
    
    # Get current deployment
    deployment = apps_v1.read_namespaced_deployment(
        name=deployment_name,
        namespace=namespace
    )
    
    # Update image
    deployment.spec.template.spec.containers[0].image = image
    
    # Apply update
    apps_v1.patch_namespaced_deployment(
        name=deployment_name,
        namespace=namespace,
        body=deployment
    )
    
    # Wait for rollout
    import time
    while True:
        deployment = apps_v1.read_namespaced_deployment(
            name=deployment_name,
            namespace=namespace
        )
        if deployment.status.updated_replicas == deployment.spec.replicas:
            if deployment.status.available_replicas == deployment.spec.replicas:
                break
        time.sleep(5)
```

## Container Registry Integration

### Docker Registry

**Authentication**: Username/Password, Token

**Image Operations**:
```python
import docker

def push_image(image_name, tag, registry):
    client = docker.from_env()
    
    # Build image
    image, build_logs = client.images.build(
        path='.',
        tag=f'{registry}/{image_name}:{tag}',
        rm=True,
        nocache=False
    )
    
    # Push to registry
    for line in client.images.push(
        f'{registry}/{image_name}',
        tag=tag,
        stream=True,
        decode=True
    ):
        if 'status' in line:
            print(line['status'])
    
    return image.id
```

### Harbor Registry

**Authentication**: Robot Account, CLI Secret

**Webhook Integration**:
```python
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)

@app.route('/webhook/harbor', methods=['POST'])
def harbor_webhook():
    # Verify signature
    signature = request.headers.get('X-Harbor-Signature')
    body = request.get_data()
    expected_sig = hmac.new(
        WEBHOOK_SECRET.encode(),
        body,
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(signature, expected_sig):
        return 'Invalid signature', 401
    
    event = request.json
    
    if event['type'] == 'PUSH_ARTIFACT':
        # Trigger deployment
        deploy_image(
            repository=event['event_data']['repository']['name'],
            tag=event['event_data']['resources'][0]['tag']
        )
    
    return 'OK', 200
```

## Monitoring Integration

### Prometheus Integration

**Metrics Export**:
```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# Define metrics
deployment_counter = Counter(
    'deployments_total',
    'Total number of deployments',
    ['environment', 'service', 'status']
)

deployment_duration = Histogram(
    'deployment_duration_seconds',
    'Time spent in deployment',
    ['environment', 'service']
)

active_deployments = Gauge(
    'active_deployments',
    'Number of active deployments',
    ['environment']
)

def record_deployment(environment, service, duration, status):
    deployment_counter.labels(
        environment=environment,
        service=service,
        status=status
    ).inc()
    
    deployment_duration.labels(
        environment=environment,
        service=service
    ).observe(duration)
```

**PromQL Queries**:
```promql
# Deployment success rate
sum(rate(deployments_total{status="success"}[5m])) 
/ 
sum(rate(deployments_total[5m]))

# Average deployment duration
avg(deployment_duration_seconds) by (environment)

# Deployments per hour
rate(deployments_total[1h]) * 3600
```

### Datadog Integration

**APM Tracing**:
```python
from ddtrace import tracer

@tracer.wrap(service='deployment-architect', resource='deploy')
def deploy_service(service_name, version):
    span = tracer.current_span()
    span.set_tag('service.name', service_name)
    span.set_tag('service.version', version)
    
    try:
        # Deployment logic
        result = execute_deployment(service_name, version)
        span.set_tag('deployment.status', 'success')
        return result
    except Exception as e:
        span.set_tag('deployment.status', 'failed')
        span.set_tag('error.message', str(e))
        raise
```

**Event API**:
```python
from datadog import initialize, api

initialize(api_key=API_KEY, app_key=APP_KEY)

def send_deployment_event(service, version, environment, status):
    api.Event.create(
        title=f'Deployment: {service} v{version}',
        text=f'Deployed {service} version {version} to {environment}',
        tags=[
            f'service:{service}',
            f'version:{version}',
            f'environment:{environment}',
            f'status:{status}'
        ],
        alert_type='info' if status == 'success' else 'error'
    )
```

## Secret Management Integration

### HashiCorp Vault

**Authentication**: AppRole, Kubernetes Auth, Token

**Secret Retrieval**:
```python
import hvac

def get_secrets(vault_addr, role_id, secret_id, path):
    client = hvac.Client(url=vault_addr)
    
    # Authenticate with AppRole
    client.auth.approle.login(
        role_id=role_id,
        secret_id=secret_id
    )
    
    # Read secrets
    secret = client.secrets.kv.v2.read_secret_version(
        path=path,
        mount_point='secret'
    )
    
    return secret['data']['data']
```

**Dynamic Database Credentials**:
```python
def get_db_credentials(vault_client, role_name):
    response = vault_client.secrets.database.generate_credentials(
        name=role_name,
        mount_point='database'
    )
    
    return {
        'username': response['data']['username'],
        'password': response['data']['password'],
        'lease_id': response['lease_id'],
        'lease_duration': response['lease_duration']
    }
```

### AWS Secrets Manager

```python
import boto3
import json

def get_secret(secret_name, region='us-east-1'):
    client = boto3.client('secretsmanager', region_name=region)
    
    response = client.get_secret_value(SecretId=secret_name)
    
    if 'SecretString' in response:
        return json.loads(response['SecretString'])
    else:
        return base64.b64decode(response['SecretBinary'])
```

## Notification Integration

### Slack

**Webhook Integration**:
```python
import requests

def send_slack_notification(webhook_url, deployment_info):
    message = {
        'blocks': [
            {
                'type': 'header',
                'text': {
                    'type': 'plain_text',
                    'text': f'🚀 Deployment: {deployment_info["service"]}'
                }
            },
            {
                'type': 'section',
                'fields': [
                    {'type': 'mrkdwn', 'text': f'*Environment:*\n{deployment_info["environment"]}'},
                    {'type': 'mrkdwn', 'text': f'*Version:*\n{deployment_info["version"]}'},
                    {'type': 'mrkdwn', 'text': f'*Status:*\n{deployment_info["status"]}'},
                    {'type': 'mrkdwn', 'text': f'*Duration:*\n{deployment_info["duration"]}s'}
                ]
            },
            {
                'type': 'actions',
                'elements': [
                    {
                        'type': 'button',
                        'text': {'type': 'plain_text', 'text': 'View Logs'},
                        'url': deployment_info['logs_url']
                    },
                    {
                        'type': 'button',
                        'text': {'type': 'plain_text', 'text': 'Rollback'},
                        'url': deployment_info['rollback_url'],
                        'style': 'danger'
                    }
                ]
            }
        ]
    }
    
    requests.post(webhook_url, json=message)
```

## Database Migration Integration

### Flyway

```python
import subprocess

def run_migrations(jdbc_url, username, password, locations):
    result = subprocess.run([
        'flyway',
        f'-url={jdbc_url}',
        f'-user={username}',
        f'-password={password}',
        f'-locations=filesystem:{locations}',
        'migrate'
    ], capture_output=True, text=True)
    
    if result.returncode != 0:
        raise Exception(f'Migration failed: {result.stderr}')
    
    return result.stdout
```

### Liquibase

```python
def run_liquibase(changelog, database_url, username, password):
    result = subprocess.run([
        'liquibase',
        f'--changelog-file={changelog}',
        f'--url={database_url}',
        f'--username={username}',
        f'--password={password}',
        'update'
    ], capture_output=True, text=True)
    
    if result.returncode != 0:
        raise Exception(f'Liquibase failed: {result.stderr}')
    
    return result.stdout
```

## API Endpoints

### Deployment API

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class DeploymentRequest(BaseModel):
    service: str
    version: str
    environment: str
    strategy: str = 'rolling'

@app.post('/deployments')
async def create_deployment(request: DeploymentRequest):
    deployment_id = generate_deployment_id()
    
    # Queue deployment
    queue_deployment(
        deployment_id=deployment_id,
        service=request.service,
        version=request.version,
        environment=request.environment,
        strategy=request.strategy
    )
    
    return {
        'deployment_id': deployment_id,
        'status': 'queued',
        'service': request.service,
        'version': request.version
    }

@app.get('/deployments/{deployment_id}')
async def get_deployment_status(deployment_id: str):
    status = fetch_deployment_status(deployment_id)
    
    if not status:
        raise HTTPException(status_code=404, detail='Deployment not found')
    
    return status

@app.post('/deployments/{deployment_id}/rollback')
async def rollback_deployment(deployment_id: str):
    result = execute_rollback(deployment_id)
    return {'status': 'rolling_back', 'rollback_id': result['id']}
```

## Integration Testing

### End-to-End Test Example

```python
import pytest
import requests

def test_full_deployment_workflow():
    # 1. Trigger deployment
    response = requests.post('http://localhost:8080/deployments', json={
        'service': 'test-service',
        'version': 'v1.2.3',
        'environment': 'staging',
        'strategy': 'rolling'
    })
    
    assert response.status_code == 200
    deployment_id = response.json()['deployment_id']
    
    # 2. Wait for deployment completion
    import time
    for _ in range(60):
        status_response = requests.get(
            f'http://localhost:8080/deployments/{deployment_id}'
        )
        status = status_response.json()['status']
        
        if status in ['completed', 'failed']:
            break
        
        time.sleep(5)
    
    assert status == 'completed'
    
    # 3. Verify service health
    health_response = requests.get('http://test-service.staging/health')
    assert health_response.status_code == 200
    assert health_response.json()['version'] == 'v1.2.3'
```

## Integration Checklist

- [ ] Source control webhooks configured
- [ ] Cloud provider credentials validated
- [ ] Kubernetes cluster access verified
- [ ] Container registry authentication working
- [ ] Monitoring exporters deployed
- [ ] Secret management integration tested
- [ ] Notification channels configured
- [ ] Database migration tool installed
- [ ] Health check endpoints validated
- [ ] Rollback procedures tested
- [ ] Documentation updated
- [ ] Team training completed
