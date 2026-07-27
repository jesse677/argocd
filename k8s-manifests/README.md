# K8s Manifests - ArgoCD GitOps

This repository contains Kubernetes manifests managed by ArgoCD for the K3s cluster.

## Repository Structure

```
k8s-manifests/
├── argocd-apps/          # ArgoCD Application definitions
│   ├── app-of-apps.yaml      # Root app (deploy this manually once)
│   ├── infrastructure.yaml   # ELK stack application
│   └── python-apps.yaml      # Python applications
│
├── infrastructure/       # Platform services
│   └── elk-stack/           # Elasticsearch, Kibana, Logstash, Filebeat
│       ├── namespace.yaml
│       ├── elasticsearch.yaml
│       ├── kibana.yaml
│       ├── logstash.yaml
│       └── filebeat.yaml
│
└── apps/                # Application workloads (empty, populated via python-apps ApplicationSet)
```

## Prerequisites

1. K3s cluster running (3 nodes)
2. ArgoCD installed via Terragrunt
3. `local-path` StorageClass available (k3s default, node-local storage)
4. Git repository pushed to GitHub/GitLab

## Initial Setup

### 1. Update Repository URLs

Before deploying, update the `repoURL` in these files to match your Git repository:

- `argocd-apps/app-of-apps.yaml`
- `argocd-apps/infrastructure.yaml`
- `argocd-apps/python-apps.yaml`

Change `https://github.com/jesse677/argocd.git` to your actual repo URL.

### 2. Deploy the App of Apps

This is the **only manual step**. Deploy the root ArgoCD Application:

```bash
kubectl apply -f argocd-apps/app-of-apps.yaml
```

### 3. Watch ArgoCD Deploy Everything

```bash
# Watch applications
kubectl get applications -n argocd -w

# Check application status
kubectl get applications -n argocd

# Check pods
kubectl get pods -n elk
```

## Accessing Services

### ArgoCD UI

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access at https://localhost:8080
# Username: admin
# Password: (from above command)
```

### Kibana UI

```bash
kubectl port-forward svc/kibana -n elk 5601:5601

# Access at http://localhost:5601
```

## How It Works

1. **App of Apps Pattern**: The root `app-of-apps.yaml` watches the `argocd-apps/` directory
2. **Automatic Sync**: ArgoCD automatically deploys any changes pushed to Git
3. **Self-Healing**: If resources are manually deleted, ArgoCD recreates them
4. **Pruning**: If you remove a manifest from Git, ArgoCD deletes it from the cluster

## Making Changes

### Add a New Application

1. Create manifests in `apps/my-new-app/`
2. Add to the ApplicationSet in `argocd-apps/python-apps.yaml`:
   ```yaml
   - app: my-new-app
     namespace: my-new-app
   ```
3. Commit and push - ArgoCD deploys automatically

### Update Existing Application

1. Edit the YAML files in Git
2. Commit and push
3. ArgoCD syncs within 3 minutes (or click "Sync" in UI)

### Remove an Application

1. Remove from the ApplicationSet list
2. Delete the manifest directory
3. Commit and push
4. ArgoCD prunes the resources

## ELK Stack Details

- **Elasticsearch**: 3 replicas, 30Gi storage each (local-path)
- **Kibana**: Single replica, visualization UI
- **Logstash**: Processes logs from Filebeat
- **Filebeat**: DaemonSet collecting container logs

Logs flow: Container stdout/stderr → Filebeat → Logstash → Elasticsearch → Kibana

## Troubleshooting

### Application not syncing

```bash
# Check application status
kubectl describe application <app-name> -n argocd

# Force sync
kubectl patch application <app-name> -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
```

### Elasticsearch pods pending

Check if the storage class exists and the PVC bound:

```bash
kubectl get storageclass
kubectl get pvc -n elk
```

### Filebeat not collecting logs

Check permissions:

```bash
kubectl logs -n elk daemonset/filebeat
```

## Next Steps

1. Replace placeholder Python app images with your actual applications
2. Setup CI pipeline to build and push container images
3. Consider adding:
   - Ingress controller for external access
   - Cert-manager for SSL certificates
   - Prometheus/Grafana for metrics
   - Secrets management (Sealed Secrets, External Secrets Operator)
