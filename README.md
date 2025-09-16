# github-actions-k8s

- Running `kubectl` against a K8s cluster using GitHub Actions
- This example runs a minikube cluster inside a VM managed by incus, then runs a `kubectl apply` against that cluster from a Github Actions runner
  - A publicly accessibly IP address/domain is required

- First up, read through/follow [manual-remote-access.md](./manual-remote-access.md) for setting up the cluster and general knowledge remote `kubectl` management of clusters
