# Linkerd + cert-manager Deployment via Argo CD (GitOps Guide)

This guide walks through deploying **Linkerd** service mesh and **cert-manager** using their **community Helm charts** via **Argo CD**, with all TLS certificates managed by cert-manager.

---

## 🔧 Prerequisites

- Kubernetes cluster with Argo CD installed
- `kubectl`, `helm`, and `argocd` CLIs configured
- Argo CD connected to Git repo hosting the manifests below

---

## 1. Install `cert-manager` using Argo CD

**File**: `apps/cert-manager.yaml`

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
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 2. Create Cert-Issuer Resources for Linkerd

**File**: `manifests/linkerd/issuer.yaml`

```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: linkerd-selfsigned
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
  commonName: "identity.linkerd.cluster.local"
  secretName: linkerd-trust-anchor
  issuerRef:
    name: linkerd-selfsigned
    kind: ClusterIssuer

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: linkerd-identity-issuer
spec:
  ca:
    secretName: linkerd-trust-anchor
```

---

## 3. Deploy Linkerd using Argo CD with cert-manager Integration

**File**: `apps/linkerd.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: linkerd-control-plane
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://helm.linkerd.io/stable
    chart: linkerd-control-plane
    targetRevision: 1.8.0
    helm:
      valueFiles:
        - values/linkerd-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: linkerd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**File**: `values/linkerd-values.yaml`

```yaml
identity:
  externalCA: true
  issuer:
    tls:
      crtPEM:
        secretName: linkerd-identity-issuer

installNamespace: true
```

---

## 4. Optional: Linkerd Viz Component

**File**: `apps/linkerd-viz.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: linkerd-viz
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://helm.linkerd.io/stable
    chart: linkerd-viz
    targetRevision: 30.8.0
  destination:
    server: https://kubernetes.default.svc
    namespace: linkerd-viz
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 5. GitOps Folder Structure

```bash
repo-root/
├── apps/
│   ├── cert-manager.yaml
│   ├── linkerd.yaml
│   └── linkerd-viz.yaml
├── manifests/
│   └── linkerd/
│       └── issuer.yaml
└── values/
    └── linkerd-values.yaml
```

This structure supports GitOps using Argo CD, allowing declarative infrastructure provisioning and automated syncing.

