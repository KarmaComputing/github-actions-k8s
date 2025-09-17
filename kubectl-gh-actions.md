- Assuming you followed [manual-remote-access.md](./manual-remote-access.md), you should now have remote access to your minikube cluster set up

# Steps

1. Add secrets for the server address and JWT token

  - Note that generated JWT tokens are relatively short-lived, but you can extend their validity by passing `--duration=<timespan>` to `kubectl create token`
    - e.g. `kubectl create-token remote-dev --duration=12h` for a token valid for 12 hours
    - We probably don't want to use these in production, your kubernetes provider (e.g. EKS) may offer a better means of authentication

  - On the webpage for your repo:
    - Settings -> Secrets and Variables -> Actions -> New Repository Secret
    - Set the name to `JWT_AUTH_TOKEN`
    - Set the value to the JWT token you generated
  - Add another secret called `API_SERVER_ADDR` with the value of your public-facing API server address

2. Access the secret in the action

  - Github actions can access repository secrets using the syntax `${{ secrets.<secret> }}`
  - We'll create a step in our action that sets the correct kubeconfig

  ```yaml
    # Other steps... #

    - name: Set kubeconfig with kubectl
      run: |
        kubectl config set-cluster "minikube" --server "${{ secrets.API_SERVER_ADDR }}"
        kubectl config set-credentials "remote-dev" --token "${{ secrets.JWT_AUTH_TOKEN }}"
        kubectl config set-context "remote-context" --cluster "minikube" --user "remote-dev"
        kubectl config use-context "remote-context"

      # kubectl command steps ... #
  ```

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
    push:
      branches:
        - "1-github-runner-manages-remote"
  jobs:
    deploy:
      runs-on: ubuntu-latest
      steps:
        - name: Install kubectl
          run: |
            mkdir $HOME/bin
            curl -Lf 'https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl' -o $HOME/bin/kubectl
            chmod +x $HOME/bin/kubectl
            echo "$HOME/bin" >> $GITHUB_PATH

        - name: Check kubectl is available on PATH
          run: kubectl version --client

        - name: Checkout repo
          uses: actions/checkout@v4

        - name: Set kubeconfig with kubectl
          run: |
            kubectl config set-cluster "minikube" --server "${{ secrets.API_SERVER_ADDR }}"
            kubectl config set-credentials "remote-dev" --token "${{ secrets.JWT_AUTH_TOKEN }}"
            kubectl config set-context "remote-context" --cluster "minikube" --user "remote-dev"
            kubectl config use-context "remote-context"

        - name: Run kubectl command against remote API
          run: kubectl get namespaces
  ```
