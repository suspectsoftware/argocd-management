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

Since external secrets is used to retrieve all sensitive data in the cluster, we need to setup the secret to access it.
In this project we use 1Password which does not have OIDC support, so we need to create the `eso-1password-credentials` secret.
We can later loop back and manage this secret with external-secrets to make rotation easier.

```
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: eso-1password-credentials
  namespace: external-secrets
type: Opaque
stringData:
  token: <1PASSWORD_SA_TOKEN>
EOF
```

And we can then apply the root application which in turn creates all underlying applications

```
kubectl apply -f root/root.yaml
```


apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: argocd-secret
  namespace: argocd
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: cluster-store
  target:
    name: argocd-secret
    labels:
      app.kubernetes.io/name: argocd-secret
      app.kubernetes.io/part-of: argocd
    template:
      type: Opaque
      data:
        server.secretkey: "{{ .SecretKey | toString }}"
        dex.google.clientID: "{{ .clientID | toString }}"
        dex.google.clientSecret: "{{ .clientSecret | toString }}"
  data:
    - secretKey: SecretKey
      remoteRef:
        key: argocd-server-secretkey/secretkey
    - secretKey: clientID
      remoteRef:
        key: google-workspace-argo-oidc/client-id
    - secretKey: clientSecret
      remoteRef:
        key: google-workspace-argo-oidc/client-secret

kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
stringData:
  server.secretkey: REDACTED
  dex.google.clientID: 397131616114-9590v8jefbn29td8kft38mtm918pb8ll.apps.googleusercontent.com
  dex.google.clientSecret: REDACTED
EOF