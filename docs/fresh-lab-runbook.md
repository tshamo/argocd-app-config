# Fresh Lab Runbook

Use this when the kind clusters or Argo CD install get into a bad state.

## Goal

Build the lab in this order:

1. `dev` cluster
2. deploy `hello-world-dev`
3. `staging` cluster
4. deploy `hello-world-staging`
5. `prod` cluster
6. deploy `hello-world-prod`

Argo CD runs in the `dev` cluster and manages all three clusters.

## Start Clean

```sh
kind delete cluster --name dev || true
kind delete cluster --name staging || true
kind delete cluster --name prod || true
```

Confirm the VM clock is correct before creating clusters:

```sh
date -u
timedatectl
```

If `System clock synchronized` is `no`, fix time before continuing. Bad time causes TLS and image-pull failures.

## Build Dev First

```sh
kind create cluster --name dev
kubectl config use-context kind-dev
kubectl get pods -n kube-system
```

## Rootless Podman Note

This lab is currently using kind with the experimental rootless Podman provider. Keep dev simple at first:

- Do not install `ingress-nginx` until Argo CD can deploy the app successfully.
- Use `kubectl port-forward` to test the app.
- If `argocd-repo-server` sees a `Kubernetes Ingress Controller Fake Certificate` when accessing GitHub, delete `ingress-nginx`; it is intercepting in-cluster HTTPS traffic.

Install Argo CD:

```sh
kubectl create namespace argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

If Redis fails to create its secret, create it manually:

```sh
kubectl create secret generic argocd-redis -n argocd \
  --from-literal=auth="$(openssl rand -base64 32)"

kubectl delete pod -n argocd --all
```

Wait for Argo CD:

```sh
kubectl get pods -n argocd -w
```

## Log In To Argo CD

In terminal 1:

```sh
kubectl config use-context kind-dev
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

In terminal 2:

```sh
PASSWORD="$(kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d)"

argocd login localhost:8080 \
  --username admin \
  --password "$PASSWORD" \
  --insecure
```

## Register Dev

```sh
argocd cluster add kind-dev --name dev --cluster-endpoint internal
```

Apply the Argo CD project:

```sh
cd ~/argocd-app-config
kubectl config use-context kind-dev
kubectl apply -k platform/projects
```

Create and sync only the dev app first:

```sh
argocd app create hello-world-dev \
  --repo https://github.com/tshamo/argocd-app-config.git \
  --path apps/hello-world/overlays/dev \
  --revision HEAD \
  --dest-name dev \
  --dest-namespace hello-world \
  --project lab \
  --sync-policy automated \
  --self-heal \
  --auto-prune \
  --sync-option CreateNamespace=true

argocd app sync hello-world-dev
argocd app get hello-world-dev
kubectl get pods -n hello-world --context kind-dev
```

Do not continue until dev is `Healthy` and `Synced`.

## Add Staging

```sh
kind create cluster --name staging
argocd cluster add kind-staging --name staging --cluster-endpoint internal
```

Create and sync staging:

```sh
argocd app create hello-world-staging \
  --repo https://github.com/tshamo/argocd-app-config.git \
  --path apps/hello-world/overlays/staging \
  --revision HEAD \
  --dest-name staging \
  --dest-namespace hello-world \
  --project lab \
  --sync-policy automated \
  --self-heal \
  --auto-prune \
  --sync-option CreateNamespace=true

argocd app sync hello-world-staging
argocd app get hello-world-staging
kubectl get pods -n hello-world --context kind-staging
```

Do not continue until staging is `Healthy` and `Synced`.

## Add Prod

```sh
kind create cluster --name prod
argocd cluster add kind-prod --name prod --cluster-endpoint internal
```

Create and sync prod:

```sh
argocd app create hello-world-prod \
  --repo https://github.com/tshamo/argocd-app-config.git \
  --path apps/hello-world/overlays/prod \
  --revision HEAD \
  --dest-name prod \
  --dest-namespace hello-world \
  --project lab \
  --sync-policy automated \
  --self-heal \
  --auto-prune \
  --sync-option CreateNamespace=true

argocd app sync hello-world-prod
argocd app get hello-world-prod
kubectl get pods -n hello-world --context kind-prod
```

## Switch Back To ApplicationSet

After all three direct apps work, delete the direct apps without deleting workloads:

```sh
argocd app delete hello-world-dev --cascade=false
argocd app delete hello-world-staging --cascade=false
argocd app delete hello-world-prod --cascade=false
```

Then let the ApplicationSet manage all three:

```sh
kubectl config use-context kind-dev
kubectl apply -k platform/applicationsets

kubectl get applicationsets -n argocd
kubectl get applications -n argocd
argocd app list
```

Expected apps:

```text
hello-world-dev
hello-world-staging
hello-world-prod
```
