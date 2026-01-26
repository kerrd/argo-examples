# Traefik + cert-manager + Let's Encrypt Setup

This setup deploys Traefik as an Ingress Controller with cert-manager for automated Let's Encrypt SSL certificates using Cloudflare DNS-01 challenge.

Based on: https://technotim.com/posts/kube-traefik-cert-manager-le/

## Architecture

- **Traefik**: Ingress controller with LoadBalancer IP `10.0.2.101`
- **cert-manager**: Automated certificate management
- **Let's Encrypt Staging**: Testing certificate issuance
- **Cloudflare DNS-01**: Wildcard certificate support

## Prerequisites

- Kubernetes cluster with ArgoCD installed
- DNS record: `traefik.hl.rushbtech.org` → `10.0.2.101`
- Cloudflare API token with DNS edit permissions
- Helm v4+ installed locally (for manifest generation)

## Directory Structure

```
traefik-cm/
├── traefik/
│   ├── base/
│   │   ├── kustomization.yaml
│   │   └── values-base.yaml
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── values-base.yaml (copy)
│   │   ├── values-dev.yaml
│   │   └── traefik-manifests.yaml (generated)
│   └── traefik-dev-application.yaml
├── cert-manager/
│   ├── base/
│   │   ├── kustomization.yaml
│   │   └── values-base.yaml
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── values-base.yaml (copy)
│   │   ├── cert-manager-crds.yaml
│   │   ├── cert-manager-manifests.yaml (generated)
│   │   ├── secret-cf-token.yaml
│   │   └── letsencrypt-staging.yaml
│   └── cert-manager-dev-application.yaml
└── test-nginx/
    ├── kustomization.yaml
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── test-nginx-application.yaml
```

## Manifest Generation

Due to Helm v4 incompatibility with Kustomize's Helm integration, manifests are pre-generated using `helm template`.

### Regenerate Traefik manifests (if needed):

```bash
cd traefik-cm/traefik/dev
helm template traefik traefik/traefik --namespace=traefik \
  --values=values-base.yaml --values=values-dev.yaml > traefik-manifests.yaml
```

### Regenerate cert-manager manifests (if needed):

```bash
cd traefik-cm/cert-manager/dev
helm template cert-manager jetstack/cert-manager --namespace=cert-manager \
  --values=values-base.yaml > cert-manager-manifests.yaml
```

### Update cert-manager CRDs (if needed):

```bash
curl -sL https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.crds.yaml \
  -o traefik-cm/cert-manager/dev/cert-manager-crds.yaml
```

## Deployment Steps

### 1. Commit and push to Git

```bash
cd /Users/dkerr/git/argo-examples
git add traefik-cm/
git commit -m "Add Traefik + cert-manager with Let's Encrypt staging"
git push
```

### 2. Deploy ApplicationSet (ArgoCD manages everything)

```bash
kubectl apply -f traefik-cm/traefik-stack-applicationset.yaml
```

This ApplicationSet will deploy in order:
1. **Wave 1**: cert-manager (with CRDs)
2. **Wave 2**: Traefik
3. **Wave 3**: test-nginx

ArgoCD will automatically handle sync order and retries.

### 3. Monitor deployment in ArgoCD UI

Or via CLI:

```bash
kubectl get applications -n argocd
kubectl get applicationset -n argocd traefik-stack
```

## Verification

### Check Traefik LoadBalancer IP:

```bash
kubectl get svc -n traefik traefik
```

Expected: `EXTERNAL-IP` should be `10.0.2.101`

### Check cert-manager ClusterIssuer:

```bash
kubectl get clusterissuer
```

Expected: `letsencrypt-staging` should show `Ready: True`

### Check certificate request:

```bash
kubectl get certificate -A
kubectl get certificaterequest -A
kubectl describe certificate traefik-hl-rushbtech-org-tls -n default
```

### Check cert-manager logs:

```bash
kubectl logs -n cert-manager -l app=cert-manager -f
```

### Test the website:

```bash
curl -k https://traefik.hl.rushbtech.org
```

Expected: Nginx welcome page with staging certificate

### Check certificate details:

```bash
openssl s_client -connect traefik.hl.rushbtech.org:443 -servername traefik.hl.rushbtech.org </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject
```

Expected issuer: `Fake LE Intermediate X1` (staging)

## Switch to Production Let's Encrypt

Once verified with staging, update to production:

1. Create `traefik-cm/cert-manager/dev/letsencrypt-production.yaml`:

```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: dkerr@rushbtech.org
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - dns01:
          cloudflare:
            email: dkerr@rushbtech.org
            apiTokenSecretRef:
              name: cloudflare-token-secret
              key: cloudflare-token
        selector:
          dnsZones:
            - "hl.rushbtech.org"
```

2. Add to kustomization.yaml
3. Update Ingress annotation to `cert-manager.io/cluster-issuer: letsencrypt-production`
4. Delete existing certificate secret: `kubectl delete secret traefik-hl-rushbtech-org-tls -n default`

## Configuration

### Traefik Settings (values-base.yaml):

- HTTP → HTTPS redirect enabled
- IngressClass: `traefik-external`
- LoadBalancer IP: `10.0.2.101`
- Debug logging enabled in dev

### cert-manager Settings (values-base.yaml):

- DNS resolvers: `1.1.1.1`, `9.9.9.9`
- CRDs installed separately
- Cloudflare DNS-01 challenge

### Cloudflare Token:

Located in: `traefik-cm/cert-manager/dev/secret-cf-token.yaml`

**⚠️ Security Note**: Consider using SealedSecrets or external secret management for production.

## Troubleshooting

### Certificate not issuing:

```bash
kubectl describe certificaterequest -n default
kubectl get challenges -A
kubectl describe order -n default <order-name>
```

### Traefik not routing:

```bash
kubectl get ingress -A
kubectl logs -n traefik -l app.kubernetes.io/name=traefik
```

### DNS challenges failing:

- Verify Cloudflare token has correct permissions
- Check cert-manager can resolve DNS: `kubectl logs -n cert-manager -l app=cert-manager`
- Verify DNS propagation: `dig @1.1.1.1 _acme-challenge.traefik.hl.rushbtech.org TXT`

## Resources

- [TechnoTim Guide](https://technotim.com/posts/kube-traefik-cert-manager-le/)
- [Traefik Docs](https://doc.traefik.io/traefik/)
- [cert-manager Docs](https://cert-manager.io/docs/)
- [Cloudflare API Tokens](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/#api-tokens)
