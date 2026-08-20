# Argo CD Management

GitOps configuration for the Argo CD cluster.

## Bootstrap

Run these commands from the repository root.

### 1. Install Argo CD

```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 9.3.4 \
  --timeout 600s \
  -f config/cluster-charts/argocd.yaml

kubectl rollout status deployment/argocd-server -n argocd --timeout=600s
```

### 2. Add GitHub credentials

The GitHub App must have access to the repositories listed in `config/repositories`.
Create its private key Secret manually. Repository ExternalSecrets read this Secret
through the in-cluster `github-app-store`. Bootstrap the management repository
directly so Argo CD can start reading this repository.

```sh
kubectl create secret generic suspectsoftware-argocd-management \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/suspectsoftware/argocd-management.git \
  --from-literal=githubAppID=2678288 \
  --from-literal=githubAppInstallationID=104764666 \
  --from-file=githubAppPrivateKey=/path/to/github-app-private-key.pem \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl label secret suspectsoftware-argocd-management \
  --namespace argocd argocd.argoproj.io/secret-type=repository --overwrite

kubectl create secret generic argocd-github-app \
  --namespace argocd \
  --from-file=githubAppPrivateKey=/path/to/github-app-private-key.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 3. Add 1Password credentials

```sh
kubectl create namespace external-secrets --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic eso-1password-credentials \
  --namespace external-secrets \
  --from-literal=token='<1PASSWORD_SA_TOKEN>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 4. Start GitOps

The project and cluster registration are required before the root Application.

```sh
kubectl apply -f config/projects/in-cluster.yaml
kubectl apply -n argocd -f config/cluster-auth/in-cluster.yaml
kubectl apply -f config/root/root.yaml
```

## Add A Repository

Add a complete repository `ExternalSecret` YAML file directly under
`config/repositories`. The `repositories` ApplicationSet creates one Application
per file.

## Add A Cluster Chart

Add the chart to `config/cluster-charts/_installed.yaml` and add a matching
`config/cluster-charts/<releaseName>.yaml` values file.
