- Assuming you followed [manual-remote-access.md](./manual-remote-access.md), you should now have remote access to your minikube cluster set up

# Steps

1. Add secrets for the server address and JWT token, and other variables

- Note that generated JWT tokens are relatively short-lived, but you can extend their validity by passing `--duration=<timespan>` to `kubectl create token`
    - e.g. `kubectl create-token remote-dev --duration=12h` for a token valid for 12 hours
    - We probably don't want to use these in production, your kubernetes provider (e.g. EKS) may offer a better means of authentication

- On the webpage for your repo:
    - Settings -> Secrets and Variables -> Actions -> New Repository Secret
    - Set the name to `KUBE_JWT_AUTH_TOKEN`
    - Set the value to the JWT token you generated
- Add another secret called `KUBE_API_SERVER_ADDR` with the value of your public-facing API server address

- We'll also add some variables for the cluster, remote username, and remote context
- On the webpage for your repo:
    - Settings -> Secrets and Variables -> Actions -> Variables -> New Repository Variable
    - Add three variables with the names and values:
        - KUBE_REMOTE_CLUSTER = minikube
        - KUBE_REMOTE_USER = remote-dev
        - KUBE_REMOTE_CONTEXT = remote-context

2. Access the secrets and variables in the action

- Github actions can access repository secrets using the syntax `${{ secrets.<secret> }}` and variables with `${{ vars.<variable> }}`
- We'll create a step in our action that sets the correct kubeconfig

```yaml
# Other steps... #

- name: Set kubeconfig with kubectl
  run: |
      kubectl config set-cluster "${{ vars.KUBE_REMOTE_CLUSTER }}" --server "${{ secrets.KUBE_API_SERVER_ADDR }}"
      kubectl config set-credentials "${{ vars.KUBE_REMOTE_USER }}" --token "${{ secrets.KUBE_JWT_AUTH_TOKEN }}"
      kubectl config set-context "${{ vars.KUBE_REMOTE_CONTEXT }}" --cluster "${{ vars.KUBE_REMOTE_CLUSTER }}" --user "${{ vars.KUBE_REMOTE_USER }}"
      kubectl config use-context "${{ vars.KUBE_REMOTE_CONTEXT }}"

# kubectl command steps ... #
```

- Using these variables and secrets makes it easier to update them in the future, without modifying the workflow file directly

3. Create the full workflow
    - So we need to:
        1. Make sure the `kubectl` binary is available
        2. Checkout the repo
        3. Configure authentication with kubectl
        4. Run `kubectl` commands against the remote API

```yaml
# File: .github/workflows/kubectl.yaml

name: Run kubectl against remote cluster
on:
  workflow_dispatch: # Allows manual start of workflows
  push:
    branches:
      - "main"
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Install kubectl
        run: |
          mkdir "$HOME/bin"
          curl -Lf "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" -o "$HOME/bin/kubectl"
          chmod +x $HOME/bin/kubectl
          echo "$HOME/bin" >> $GITHUB_PATH

      - name: Check kubectl is available on PATH
        run: kubectl version --client

      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Set kubeconfig with kubectl
        run: |
          kubectl config set-cluster "${{ vars.KUBE_REMOTE_CLUSTER }}" --server "${{ secrets.KUBE_API_SERVER_ADDR }}"
          kubectl config set-credentials "${{ vars.KUBE_REMOTE_USER }}" --token "${{ secrets.KUBE_JWT_AUTH_TOKEN }}"
          kubectl config set-context "${{ vars.KUBE_REMOTE_CONTEXT }}" --cluster "${{ vars.KUBE_REMOTE_CLUSTER }}" --user "${{ vars.KUBE_REMOTE_USER }}"
          kubectl config use-context "${{ vars.KUBE_REMOTE_CONTEXT }}"

      - name: Run kubectl command against remote API
        run: kubectl get namespaces
```

- With this, we have remote access to the API in according with the RBAC rules we created earlier
- If you wanted to `kubectl apply -f` in this action, you could do so like below:

```yaml
# previous setup steps #

- name: kubectl apply with a file
  run: kubectl apply -f "${GITHUB_WORKSPACE}/manifests/nginx-test.yml"
```

3. Make this a bit more robust

    1. First up we can separate the logic for installing and configuring `kubectl` into a reusable action.

        - Separating the setup and configuration of `kubectl` into a reusable action makes it reusable across any number of workflows (duh) but also allows for controlled caching of the `kubectl` binary.
