- Assuming you followed [manual-remote-access.md](./manual-remote-access.md), you should now have remote access to your minikube cluster set up

# Steps

1. Make the JWT token accessible to the GitHub runner

  - Note that generated JWT tokens are relatively short-lived, but you can extend their validity by passing `--duration=<timespan>` to `kubectl create token`
    - e.g. `kubectl create-token remote-dev --duration=12h` for a token valid for 12 hours
    - We probably don't want to use these in production, your kubernetes provider (e.g. EKS) may offer a better means of authentication

  - On the webpage for your repo:
    - Settings -> Secrets and Variables -> Actions -> New Repository Secret
    - Set the name to `JWT_AUTH_TOKEN`
    - Set the value to the JWT token you generated
