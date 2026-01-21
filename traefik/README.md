# GitOps with ArgoCD

This directory contains ArgoCD applications using the App of Apps pattern.

## Structure

```
gitops/
├── app-of-apps/          # Root application
│   └── root.yaml         # App of Apps entry point
└── apps/                 # Individual applications
    └── traefik/          # Traefik ingress controller
        ├── application.yaml
        └── values.yaml
```

## App of Apps Pattern

The App of Apps pattern allows ArgoCD to manage multiple applications from a single root application. The root application watches the `apps/` directory and automatically creates child applications.

## Deployment

1. **Apply the root application:**
   ```bash
   kubectl apply -f gitops/app-of-apps/root.yaml
   ```

2. **Verify deployment:**
   ```bash
   kubectl get applications -n argocd
   kubectl get pods -n traefik
   ```

## Applications

### Traefik

Traefik is deployed as an ingress controller using the official Helm chart.

**Configuration:** [apps/traefik/values.yaml](apps/traefik/values.yaml)

**Features:**
- LoadBalancer service type
- HTTP (80) and HTTPS (443) ports
- Dashboard enabled
- Default ingress class
- Security context configured

**Customize:**
- Edit `values.yaml` to set LoadBalancer IP, enable Let's Encrypt, etc.
- Commit changes and ArgoCD will auto-sync

## Adding New Applications

1. Create a new directory under `apps/`:
   ```bash
   mkdir -p gitops/apps/my-app
   ```

2. Create an `application.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: my-app
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: <chart-repo-or-git-repo>
       targetRevision: <version>
       # ... chart or path config
     destination:
       server: https://kubernetes.default.svc
       namespace: my-app
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```

3. Commit and push - ArgoCD will automatically detect and deploy it.

## Notes

- Update the `repoURL` in [app-of-apps/root.yaml](app-of-apps/root.yaml) to match your Git repository
- The root application uses automated sync with pruning and self-healing
- All child applications inherit similar sync policies
