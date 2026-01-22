# Nginx GitOps Deployment

This directory contains Nginx deployments using Kustomize + Helm for ArgoCD, following the pattern from [this article](https://medium.com/@nsalexamy/using-kustomize-with-helm-charts-for-argo-cd-applications-0c87132d296c).

## Structure

```
nginx/
├── argocd/                          # ArgoCD application manifests
│   ├── nginx-dev-application.yaml   # Dev application (individual)
│   ├── nginx-prod-application.yaml  # Prod application (individual)
│   └── nginx-applicationset.yaml    # ApplicationSet (manages both)
├── dev/                             # Development environment
│   ├── kustomization.yaml           # Kustomize config with Helm chart
│   └── values-dev.yaml              # Complete Helm values for dev
└── prod/                            # Production environment
    ├── kustomization.yaml           # Kustomize config with Helm chart
    └── values-prod.yaml             # Complete Helm values for prod
```

## Key Principles

### 1. Helm Template + Kustomize

Each environment uses Kustomize's `helmCharts` feature to inflate the Helm chart:

```yaml
helmCharts:
  - name: nginx
    repo: https://charts.bitnami.com/bitnami
    version: 18.2.4
    valuesFile: values-{env}.yaml
```

This generates manifests from the Helm chart, which Kustomize then processes.

### 2. Complete Values Per Environment

Each environment has its own complete `values-{env}.yaml` file. We don't merge values - each file contains all necessary configuration for that environment.

**Dev characteristics:**
- 1 replica
- Lower resources (100m CPU, 128Mi memory)
- ClusterIP service
- Hostname: `nginx-dev.hl.rushbtech.org`

**Prod characteristics:**
- 2 replicas (HA)
- Higher resources (200m CPU, 256Mi memory)
- Autoscaling enabled (2-5 pods)
- Pod anti-affinity for distribution
- Pod disruption budget
- Hostname: `nginx.hl.rushbtech.org`

### 3. No Base Directory

Following the article's guidance: we don't use a `base/` directory because we don't have shared Kubernetes resources. Each environment directly references the Helm chart with its own values.

## Deployment Options

### Option 1: Individual Applications

Deploy one environment at a time:

```bash
# Deploy dev
kubectl apply -f argocd/nginx-dev-application.yaml

# Deploy prod
kubectl apply -f argocd/nginx-prod-application.yaml
```

### Option 2: ApplicationSet (Recommended)

Deploy both environments with a single manifest:

```bash
kubectl apply -f argocd/nginx-applicationset.yaml
```

This creates two applications: `nginx-dev` and `nginx-prod`.

## Prerequisites

### ArgoCD Configuration

ArgoCD must be configured to enable Helm chart inflation:

```bash
# Add --enable-helm to ArgoCD ConfigMap
kubectl patch configmap argocd-cm -n argocd --type merge \
  -p '{"data":{"kustomize.buildOptions":"--enable-helm"}}'

# Restart repo server
kubectl rollout restart deployment argocd-repo-server -n argocd
```

### Repository URL

Update the `repoURL` in the ArgoCD application manifests to match your Git repository:

```yaml
source:
  repoURL: https://github.com/dkerr-org/homelab.git  # Change this
```

## Testing Locally

Before deploying with ArgoCD, test the Kustomize build:

```bash
# Build dev environment
kustomize build dev --enable-helm

# Build prod environment
kustomize build prod --enable-helm

# Apply directly to cluster (testing)
kustomize build dev --enable-helm | kubectl apply -f -
```

## Monitoring

After deployment, verify the applications:

```bash
# Check ArgoCD applications
kubectl get applications -n argocd | grep nginx

# Check pods
kubectl get pods -n nginx-dev
kubectl get pods -n nginx-prod

# Check ingress
kubectl get ingress -n nginx-dev
kubectl get ingress -n nginx-prod
```

## Customization

To modify environment configuration:

1. Edit the appropriate `values-{env}.yaml` file
2. Commit and push to Git
3. ArgoCD will automatically sync (or manually sync if needed)

Example customizations:
- Change replica counts
- Adjust resource limits
- Modify ingress hostnames
- Add custom nginx configuration
- Enable/disable features

## Promotion Workflow

To promote changes from dev to prod:

1. Test in dev environment first
2. Review the changes in `values-dev.yaml`
3. Copy relevant changes to `values-prod.yaml`
4. Commit and push
5. ArgoCD syncs prod automatically

## Notes

- Ingress configured for Traefik with Let's Encrypt
- TLS enabled on both environments
- Health checks configured (liveness and readiness probes)
- Production has HA configuration with anti-affinity
- Both environments use ClusterIP services (exposed via Ingress)
