# ArgoCD Management
This repository contains the management state for the ArgoCD cluster.

## Bootstrapping ArgoCD
To bootstrap the cluster we first need to install our specific version of ArgoCD using Helm.

```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd --namespace argocd --version 9.3.4 -f argocd-values.yaml --timeout 600s --create-namespace --debug
```

After installing ArgoCD we can create the management GitHub repository secret.

```
kubectl apply -f bootstrap/repository-secret.yaml
```

And we can then apply the root application which in turn creates all underlying applications

```
kubectl apply -f root/root.yaml
```
