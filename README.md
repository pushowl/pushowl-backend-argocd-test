# PushOwl Backend - ArgoCD Helm Charts

This repository contains Helm charts for deploying the PushOwl backend application using ArgoCD on Minikube (local testing).

## Repository Structure

```
.
├── argocd-application.yaml          # ArgoCD Application definition
└── pushowl-backend/                 # Helm chart for PushOwl backend
    ├── Chart.yaml
    ├── values.yaml                  # Default values
    ├── values-minikube.yaml         # Minikube-specific values
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        └── configmap.yaml
```

## Prerequisites

1. **Minikube** running with ArgoCD installed
2. **Docker** for building images
3. **kubectl** configured to use Minikube context
4. **Docker Compose services** running (PostgreSQL, Redis, RabbitMQ, etc.)

## Initial Setup

### 1. Verify Minikube Context

```bash
kubectl config current-context
# Should show: minikube
```

### 2. Start Docker Compose Services

```bash
# In pushowl-backend repo
docker-compose up -d

# Verify services are running
docker ps --format "table {{.Names}}\t{{.Status}}"
```

### 3. Deploy Application via ArgoCD

```bash
# Apply ArgoCD application
kubectl apply -f argocd-application.yaml

# Watch deployment
kubectl get application pushowl-backend-minikube -n argocd -w

# Check pods
kubectl get pods -n pushowl-backend
```

### 4. Access the Application

```bash
# Port-forward to localhost
kubectl port-forward -n pushowl-backend svc/pushowl-backend 8000:80

# Test healthcheck
curl http://localhost:8000/healthcheck/
```

## Local Development Workflow

### Making Changes and Deploying

When you make code changes in the backend and want to deploy them:

#### Step 1: Build the Backend Image

```bash
# In pushowl-backend repo (parent directory)
cd ../

# Get current commit SHA
COMMIT_SHA=$(git log -1 --format="%h")
echo "Building image with tag: $COMMIT_SHA"

# Build Docker image
docker build -t localhost:5000/pushowl-backend:$COMMIT_SHA .
```

#### Step 2: Load Image into Minikube

**CRITICAL:** Minikube has its own Docker daemon. You must load the image explicitly.

```bash
# Load image into Minikube
minikube image load localhost:5000/pushowl-backend:$COMMIT_SHA

# Verify image is available in Minikube
minikube image ls | grep pushowl-backend
```

#### Step 3: Update Helm Chart

```bash
# Return to helm chart directory
cd pushowl-argocd-helm

# Edit values-minikube.yaml and update the image tag
# Change line 6: tag: "old-commit" to tag: "new-commit"

# Example using sed (replace NEW_COMMIT_SHA with your actual commit)
sed -i '' "s/tag: \".*\"/tag: \"$COMMIT_SHA\"/" pushowl-backend/values-minikube.yaml

# Commit and push changes
git add pushowl-backend/values-minikube.yaml
git commit -m "Deploy $COMMIT_SHA to minikube"
git push
```

#### Step 4: Trigger ArgoCD Sync

ArgoCD polls the Git repository every 3 minutes by default. You can either:

**Option A: Wait for auto-sync (3 minutes)**
```bash
# Watch the sync happen automatically
kubectl get application pushowl-backend-minikube -n argocd -w
```

**Option B: Manual sync (instant)**
```bash
# Trigger sync immediately
kubectl patch application pushowl-backend-minikube -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# Or using ArgoCD CLI
argocd app sync pushowl-backend-minikube
```

#### Step 5: Watch Deployment

```bash
# Watch pods rolling out
kubectl get pods -n pushowl-backend -w

# Check logs of new pod
kubectl logs -n pushowl-backend -l app.kubernetes.io/name=pushowl-backend --tail=50
```

## Complete Deployment Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Make code changes in pushowl-backend                     │
│    git commit -m "changes"                                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Build Docker image                                        │
│    docker build -t localhost:5000/pushowl-backend:$SHA .    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Load into Minikube (CRITICAL!)                           │
│    minikube image load localhost:5000/pushowl-backend:$SHA  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Update Helm chart values-minikube.yaml                   │
│    Change tag to new commit SHA                             │
│    git push to this repo                                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. ArgoCD detects change (3 min poll or manual sync)       │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. ArgoCD applies Helm chart → Rolling deployment          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. New pods running with updated code!                      │
└─────────────────────────────────────────────────────────────┘
```

## ArgoCD Configuration

The ArgoCD Application (`argocd-application.yaml`) is configured with:

- **Auto-sync enabled**: Changes are automatically deployed
- **Self-heal enabled**: Kubernetes changes are automatically reverted to Git state
- **Prune enabled**: Deleted resources in Git are removed from cluster
- **Poll interval**: 3 minutes (default)

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
```

## Accessing ArgoCD UI

To view your deployment in the ArgoCD web UI:

```bash
# Port-forward ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Access UI
# URL: https://localhost:8080
# Username: admin
# Password: (from command above)
```

## Environment Configuration

The `values-minikube.yaml` file is configured to connect to Docker Compose services running on your host machine:

- **PostgreSQL**: `host.minikube.internal:5432`
- **Redis**: `host.minikube.internal:6379`
- **RabbitMQ**: `host.minikube.internal:5672`
- **DynamoDB**: `host.minikube.internal:9000`
- **ClickHouse**: `host.minikube.internal:8123`

All environment variables are stored in the ConfigMap created by the Helm chart.

## Troubleshooting

### Pod not starting

```bash
# Check pod events
kubectl describe pod -n pushowl-backend <pod-name>

# Check logs
kubectl logs -n pushowl-backend <pod-name> --tail=100
```

### Image pull errors

```bash
# Verify image exists in Minikube
minikube image ls | grep pushowl-backend

# If missing, reload the image
minikube image load localhost:5000/pushowl-backend:$COMMIT_SHA
```

### ArgoCD not syncing

```bash
# Check application status
kubectl get application pushowl-backend-minikube -n argocd -o yaml

# Force refresh
kubectl patch application pushowl-backend-minikube -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

### Cannot connect to services

Ensure Docker Compose services are running:

```bash
docker ps | grep -E "postgres|redis|rabbitmq"
```

### Readiness probe failing

The pod may take 30-60 seconds to become ready. Check logs:

```bash
kubectl logs -n pushowl-backend -l app.kubernetes.io/name=pushowl-backend
```

## Quick Reference Commands

```bash
# Check deployment status
kubectl get all -n pushowl-backend

# View application in ArgoCD
kubectl get application -n argocd

# Restart deployment
kubectl rollout restart deployment/pushowl-backend -n pushowl-backend

# Delete and redeploy
kubectl delete application pushowl-backend-minikube -n argocd
kubectl apply -f argocd-application.yaml

# View all resources managed by ArgoCD
kubectl get all -n pushowl-backend -l app.kubernetes.io/instance=pushowl-backend-minikube
```

## Production Deployment (Future)

For production deployment, you would:

1. Push Docker images to a container registry (ECR, Docker Hub, etc.)
2. Create separate `values-production.yaml` with production configurations
3. Update `argocd-application.yaml` to point to production cluster
4. Enable webhooks for instant deployments instead of 3-min polling

## Resources

- **GitHub Repo**: https://github.com/pushowl/pushowl-backend-argocd-test
- **Backend Repo**: https://github.com/pushowl/pushowl-backend
- **ArgoCD Docs**: https://argo-cd.readthedocs.io/
- **Helm Docs**: https://helm.sh/docs/