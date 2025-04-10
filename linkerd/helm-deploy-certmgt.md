
# 🚀 Deploy Linkerd with Cert Manager via Argo CD (Inline Helm Parameters)

This setup includes:
- **Cert Manager** deployed to the `cert-manager` namespace
- **Linkerd** control plane deployed to the `linkerd` namespace
- **All TLS certificates managed by Cert Manager**
- **Everything defined as Argo CD Applications**
- **No external `values.yaml` — all values set inline**

---

## ✅ Prerequisites

- Argo CD installed and running
- Cert Manager Helm repo: `https://charts.jetstack.io`
- Linkerd Helm repo: `https://helm.linkerd.io/stable`
- GitOps repo with these manifests

---

## 📁 GitOps Repository Layout

```plaintext
.
├── cert-manager
│   └── app.yaml
├── linkerd
│   ├── app-crds.yaml
│   └── app-control-plane.yaml
└── linkerd-certs
    ├── ca.yaml
    ├── issuer.yaml
    └── app.yaml
```

---

## 1. 🚀 Cert Manager (with CRDs)

### cert-manager/app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: v1.14.3
    helm:
      parameters:
        - name: installCRDs
          value: "true"
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

---

## 2. 🛡️ Linkerd Trust Anchor and Issuer (Cert Manager Certificates)

### linkerd-certs/ca.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: linkerd-trust-anchor
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-trust-anchor
  namespace: linkerd
spec:
  isCA: true
  commonName: identity.linkerd.cluster.local
  secretName: linkerd-trust-anchor
  duration: 87600h
  issuerRef:
    name: linkerd-trust-anchor
    kind: ClusterIssuer
```

### linkerd-certs/issuer.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: linkerd-issuer
  namespace: linkerd
spec:
  ca:
    secretName: linkerd-trust-anchor
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-identity-issuer
  namespace: linkerd
spec:
  commonName: identity.linkerd.cluster.local
  secretName: linkerd-identity-issuer
  duration: 43800h
  issuerRef:
    name: linkerd-issuer
    kind: Issuer
```

### linkerd-certs/app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: linkerd-certs
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://your-git-repo
    path: linkerd-certs
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: linkerd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

---

## 3. ⚙️ Linkerd CRDs (must be deployed first)

### linkerd/app-crds.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: linkerd-crds
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://helm.linkerd.io/stable
    chart: linkerd-crds
    targetRevision: 1.15.2
  destination:
    server: https://kubernetes.default.svc
    namespace: linkerd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

---

## 4. 🚀 Linkerd Control Plane with External CA (Cert Manager)

### linkerd/app-control-plane.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: linkerd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://helm.linkerd.io/stable
    chart: linkerd-control-plane
    targetRevision: 1.15.2
    helm:
      parameters:
        - name: identity.externalCA
          value: "true"
  destination:
    server: https://kubernetes.default.svc
    namespace: linkerd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

> ✅ The secret `linkerd-identity-issuer` must be present in the `linkerd` namespace before this chart is synced.

---

## 🧠 Sync Order Notes

If using Argo CD v2.7+, you can use Application dependencies (preferred):

```yaml
  dependencies:
    - cert-manager
    - linkerd-certs
    - linkerd-crds
```

Or define sync waves:

```yaml
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
    syncWave: "0" # for cert-manager
```

Then:
- `linkerd-certs` → syncWave: `"1"`
- `linkerd-crds` → syncWave: `"1"`
- `linkerd` → syncWave: `"2"`

---

Let me know if you want to extend this to include:
- Linkerd Viz
- Automatic mesh injection
- Dashboard/observability tooling
