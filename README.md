# github-actions-k8s

## A quick note

- This repo is an **example** of how to do this, and for that it uses minikube. This repo does **not** create a production ready environment, it merely displays the concepts and methods for remotely managing a cluster with kubectl.

## Get started

- Running `kubectl` against a K8s cluster using GitHub Actions
- This example runs a minikube cluster inside a VM managed by incus, then runs a `kubectl apply` against that cluster from a Github Actions runner
  - A publicly accessibly IP address/domain is required

- First up, read through/follow [manual-remote-access.md](./manual-remote-access.md) for setting up the cluster and general knowledge remote `kubectl` management of clusters
- Then, you can follow along with [kubectl-gh-actions.md](./kubectl-gh-actions.md) for setting up a github actions workflow which runs `kubectl apply` against a remote cluster
