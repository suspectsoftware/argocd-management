# ArgoCD Management
This repository contains the management state for the ArgoCD cluster.

## Bootstrapping ArgoCD
To bootstrap the cluster we first need to install our specific version of ArgoCD using Helm.

```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd \
    --namespace argocd \
    --version 9.3.4 \
    --timeout 600s \
    --create-namespace \
    --debug \
    -f config/cluster-charts/argocd.yaml
```

After installing ArgoCD we can create the management GitHub repository secret.

```
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: suspectsoftware-argocd-management
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/suspectsoftware/argocd-management.git
  githubAppID: "2678288"
  githubAppInstallationID: "104764666"
  githubAppPrivateKey: |
    -----BEGIN RSA PRIVATE KEY-----
    .,.,.,.,.,.,.,.,.,.,.,.,.,.,.,.
    -----END RSA PRIVATE KEY-----
EOF
```

And we can then apply the root application which in turn creates all underlying applications

```
kubectl apply -f root/root.yaml
```
