# VProfile Application - GitOps Kubernetes Definitions

This repository contains Kubernetes manifest files for deploying the **vprofile** multi-tier application using a **GitOps approach**. The manifests are designed to be used with GitOps tools like ArgoCD or Flux to enable automated, declarative deployments.

## 🎯 Project Overview

This project showcases GitOps best practices by maintaining Kubernetes resources as code. All application infrastructure is defined declaratively in YAML manifests, enabling:

- **Version Control**: All infrastructure changes are tracked in Git
- **Automated Deployments**: GitOps operators automatically sync cluster state with repository state
- **Rollback Capability**: Easy rollback to previous versions via Git history
- **Audit Trail**: Complete history of who changed what and when
- **Declarative Configuration**: Desired state is defined in code, not imperatively applied

## 🏗️ Application Architecture

The vprofile application consists of a multi-tier architecture:

```
┌─────────────┐
│   Ingress   │  (External Access via NGINX)
└──────┬──────┘
       │
┌──────▼──────────┐
│  VProfile App   │  (Java Application - Port 8080)
└────┬───┬───┬────┘
     │   │   │
     │   │   └─────────┐
     │   │             │
┌────▼─────┐  ┌───────▼────┐  ┌──────────┐
│  MySQL   │  │ Memcached  │  │ RabbitMQ │
│ Database │  │   Cache    │  │  Queue   │
└──────────┘  └────────────┘  └──────────┘
```

## 📦 Components

### Application Layer
- **vproapp**: Main Java application running on Tomcat
  - Docker Image: `ochukowh/vprofileapp:latest`
  - Port: 8080
  - Auto-scaling enabled via HPA
  - Init containers ensure dependencies are ready

### Data Layer
- **MySQL Database (vprodb)**: Persistent data storage
  - Docker Image: `ochukowh/vprofiledb:latest`
  - Port: 3306
  - Persistent Volume for data storage
  - Resource limits enforced

### Caching & Messaging
- **Memcached (vpromc)**: Application caching layer
  - Port: 11211
  - Optimized resource allocation

- **RabbitMQ (vpromq01)**: Message queue
  - Port: 5672
  - Used for asynchronous processing

### Infrastructure Features
- **Horizontal Pod Autoscaler (HPA)**: Auto-scales application based on CPU/Memory
- **Resource Quotas**: Namespace-level resource management
- **Limit Ranges**: Default resource limits for pods
- **Ingress**: External access with SSL redirect
- **Secrets**: Secure credential management

## 📋 Prerequisites

- Kubernetes cluster (v1.19+)
- GitOps tool (ArgoCD recommended or Flux)
- NGINX Ingress Controller installed
- Metrics Server (for HPA functionality)
- kubectl CLI tool
- Git repository access

## 🚀 Deployment with GitOps (ArgoCD)

### Quick Start: Using Provided Manifests (Recommended)

This repository includes pre-configured ArgoCD Project and Application manifests for easy deployment.

**Step 1: Update the Application manifest**

Edit `argocd-application.yaml` and update the repository URL:
```yaml
source:
  repoURL: https://github.com/YOUR-USERNAME/argo-project-defs.git
  targetRevision: main
```

**Step 2: Apply the manifests**

```bash
# Create the ArgoCD Project
kubectl apply -f argocd-project.yaml

# Create the ArgoCD Application
kubectl apply -f argocd-application.yaml

# Watch the deployment
kubectl get applications -n argocd -w
```

**Step 3: Monitor via ArgoCD UI**

Access ArgoCD UI and navigate to the `vprofile-app` application to see the sync status.

```bash
# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward to ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080
```

---

### Alternative: Manual Setup via ArgoCD UI

1. **Login to ArgoCD UI**

2. **Create New Application**:
   - Application Name: `vprofile-app`
   - Project: `default`
   - Sync Policy: `Automatic`
   - Repository URL: `<your-git-repo-url>`
   - Revision: `main` or `master`
   - Path: `.` (or subdirectory containing manifests)
   - Cluster: `in-cluster` (or target cluster)
   - Namespace: `vprofile`

3. **Enable Auto-Sync and Self-Heal** (Optional but recommended)

4. **Click Create and Sync**

### Alternative: Using ArgoCD CLI

```bash
# Create ArgoCD Project
argocd proj create vprofile-project \
  --description "VProfile Application Project" \
  --dest https://kubernetes.default.svc,vprofile

# Create ArgoCD Application
argocd app create vprofile-app \
  --project vprofile-project \
  --repo <your-git-repo-url> \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace vprofile \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Sync the application
argocd app sync vprofile-app

# Watch sync status
argocd app get vprofile-app --watch
```

## 🎯 ArgoCD Project Features

The `argocd-project.yaml` defines a dedicated ArgoCD Project with the following features:

### Resource Policies
- **Whitelisted Resources**: Only approved Kubernetes resources can be deployed
- **Cluster Resources**: Namespaces, PersistentVolumes, StorageClasses, ClusterRoles
- **Namespace Resources**: Deployments, Services, Ingress, HPA, Secrets, ConfigMaps, etc.

### Multi-Environment Support
Allows deployment to multiple namespaces:
- `vprofile` (production)
- `vprofile-staging`
- `vprofile-prod`

### RBAC Roles
Three predefined roles for access control:

| Role | Permissions | User Group |
|------|-------------|------------|
| `project-admin` | Full access to all applications | `vprofile-admins` |
| `developer` | Sync and view applications | `vprofile-developers` |
| `readonly` | Read-only access | `vprofile-viewers` |

### Sync Windows
- Allows syncs at any time by default
- Configurable maintenance windows (commented examples included)
- Manual sync always permitted

### Orphaned Resources
- Warns when resources exist in cluster but not in Git
- Helps maintain GitOps hygiene

## 📁 Manifest Files Description

### ArgoCD GitOps Manifests
| File | Description |
|------|-------------|
| `argocd-project.yaml` | ArgoCD Project definition with RBAC and resource policies |
| `argocd-application.yaml` | ArgoCD Application definition for automated deployment |

### Application Manifests
| File | Description |
|------|-------------|
| `appdeploy.yaml` | Deployment manifest for the vprofile application |
| `appservice.yaml` | ClusterIP Service for vprofile app |
| `appingress.yaml` | Ingress resource for external access |
| `dbdeploy.yaml` | MySQL database deployment with persistent storage |
| `dbservice.yaml` | ClusterIP Service for MySQL database |
| `dbpvc.yaml` | PersistentVolumeClaim for database storage |
| `mcdep.yaml` | Memcached deployment |
| `mcservice.yaml` | ClusterIP Service for Memcached |
| `rmqdeploy.yaml` | RabbitMQ deployment |
| `rmqservice.yaml` | ClusterIP Service for RabbitMQ |
| `secret.yaml` | Kubernetes Secret for database and RabbitMQ credentials |
| `hpa.yaml` | HorizontalPodAutoscaler for vproapp |
| `hpa-all.yaml` | Additional HPA configurations |
| `resource-quota.yaml` | Namespace resource quotas |
| `limit-range.yaml` | Default resource limits for pods |

## ⚙️ Configuration

### Updating Secrets

Before deployment, update the `secret.yaml` with your own base64-encoded credentials:

```bash
# Encode your database password
echo -n 'your-db-password' | base64

# Encode your RabbitMQ password
echo -n 'your-rmq-password' | base64
```

Update the `secret.yaml` file with the encoded values.

### Customizing Domain

Update the Ingress host in `appingress.yaml`:

```yaml
spec:
  rules:
  - host: your-domain.com  # Change this
```

### Adjusting Resources

Modify resource requests/limits in deployment files based on your cluster capacity:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Scaling Configuration

Adjust HPA settings in `hpa.yaml`:

```yaml
minReplicas: 1    # Minimum pod count
maxReplicas: 5    # Maximum pod count
```

## 🔍 Monitoring Deployment

```bash
# Watch ArgoCD application status
argocd app get vprofile-app

# Watch pods in vprofile namespace
kubectl get pods -n vprofile -w

# Check application logs
kubectl logs -n vprofile -l app=vproapp -f

# Verify services
kubectl get svc -n vprofile

# Check ingress
kubectl get ingress -n vprofile

# Monitor HPA status
kubectl get hpa -n vprofile -w
```

## 🔄 GitOps Workflow

1. **Make Changes**: Update manifest files in this repository
2. **Commit & Push**: Commit changes to Git
3. **Auto-Sync**: ArgoCD detects changes and syncs automatically (if auto-sync enabled)
4. **Manual Sync**: Or manually sync via ArgoCD UI/CLI
5. **Rollback**: Revert Git commit to rollback deployment

## 🧪 Verification

After deployment, verify the application:

```bash
# Check all pods are running
kubectl get pods -n vprofile

# Access the application
curl https://vprofile.ochukowhoro.xyz

# Or port-forward for testing
kubectl port-forward -n vprofile svc/vproapp-service 8080:8080
# Then access http://localhost:8080
```

## 📊 Auto-Scaling

The application includes Horizontal Pod Autoscaling:

- **Minimum Replicas**: 1
- **Maximum Replicas**: 5
- **CPU Target**: 70% utilization
- **Memory Target**: 80% utilization
- **Scale Up**: Fast (100% increase or 2 pods per 30s)
- **Scale Down**: Gradual (50% decrease per 60s with 5min stabilization)

## 🔒 Security Considerations

- Secrets are stored in Kubernetes Secrets (consider using Sealed Secrets or External Secrets Operator)
- Resource limits prevent resource exhaustion
- Network policies can be added for enhanced security
- RBAC should be configured for ArgoCD service account

## 🛠️ Troubleshooting

### Pods not starting
```bash
kubectl describe pod <pod-name> -n vprofile
kubectl logs <pod-name> -n vprofile
```

### Database connection issues
Check if init containers completed successfully and database service is accessible.

### Ingress not working
Ensure NGINX Ingress Controller is installed and DNS is properly configured.

### HPA not scaling
Verify Metrics Server is installed:
```bash
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

## 🤝 Contributing

To contribute to this project:

1. Fork the repository
2. Create a feature branch
3. Make your changes to the manifests
4. Test in a dev/staging environment
5. Submit a pull request
6. ArgoCD will automatically deploy after merge to main

## 📝 License

This project is open-source and available for educational purposes.

## 👥 Author

- Docker Images: ochukowh
- Kubernetes Manifests: GitOps-ready definitions

## 🔗 Additional Resources

- **[ArgoCD Quick Start Guide](./ARGOCD-QUICKSTART.md)** - Step-by-step commands for common operations
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [GitOps Principles](https://www.gitops.tech/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)

