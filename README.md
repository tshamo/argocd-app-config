# Argo CD Local Lab

This repo is structured for a local GitOps lab with three kind clusters:

- `dev`
- `staging`
- `prod`

The same `hello-world` app is deployed to each environment through one Argo CD ApplicationSet.

## Layout

```text
apps/
  hello-world/
    base/
    overlays/
      dev/
      staging/
      prod/
clusters/
  dev/config.yaml
  staging/config.yaml
  prod/config.yaml
platform/
  projects/
  applicationsets/
```

## How It Works

`platform/applicationsets/hello-world.yaml` reads every `clusters/*/config.yaml` file and creates one Argo CD Application per environment.

Each generated Application points to:

```text
apps/hello-world/overlays/<environment>
```

For example, `clusters/dev/config.yaml` creates `hello-world-dev` and deploys the dev overlay to the `hello-world` namespace on the `dev` cluster.

Cluster config files describe where each environment deploys:

```yaml
environment: dev
tier: nonprod
namespace: hello-world
cluster:
  name: dev
  context: kind-dev
  provider: kind
```

Because each environment has its own cluster, the namespace can stay the same in every cluster: `hello-world`.

## VM Setup

Prerequisites:

- `kubectl`
- `kind`
- `argocd`

Create the clusters:

```sh
kind create cluster --name dev
kind create cluster --name staging
kind create cluster --name prod
```

Install Argo CD on the `dev` cluster:

```sh
kubectl config use-context kind-dev
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Register all three clusters with Argo CD:

```sh
argocd cluster add kind-dev --name dev
argocd cluster add kind-staging --name staging
argocd cluster add kind-prod --name prod
```

Apply the ApplicationSet to Argo CD:

```sh
kubectl config use-context kind-dev
kubectl apply -k platform/projects
kubectl apply -k platform/applicationsets
```

Argo CD will create:

- `hello-world-dev`
- `hello-world-staging`
- `hello-world-prod`
