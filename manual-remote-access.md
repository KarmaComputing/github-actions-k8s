# Steps

1. Create and enter VM with the required resources

```shell
$ incus launch --vm -c security.secureboot=false -c limits.cpu=6 -c limits.memory=8GiB images:ubuntu/24.04 kubectl-ghactions-test
$ incus exec kubectl-ghactions-test -- su -l ubuntu
```

2. Update the VM and install docker

```shell
ubuntu@kubectl-ghactions-test:~$ sudo apt update && sudo apt upgrade
ubuntu@kubectl-ghactions-test:~$ sudo apt install docker.io
ubuntu@kubectl-ghactions-test:~$ sudo usermod -aG docker ubuntu
ubuntu@kubectl-ghactions-test:~$ newgrp docker
ubuntu@kubectl-ghactions-test:~$ sudo systemctl enable --now docker
```

3. Install minikube

```shell
sudo apt install curl # It's not preinstalled

curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64

# Install minikube binary to /usr/local/bin/minikube
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

- Check by running `minikube`

```shell
ubuntu@kubectl-ghactions-test:~$ minikube
  minikube provisions and manages local Kubernetes clusters optimized for development workflows.
  ...
```

- We'll also just use minikube's provided kubectl for testing

```shell
ubuntu@kubectl-ghactions-test:~$ alias kubectl="minikube kubectl --"
```

4. Start a cluster

```shell
ubuntu@kubectl-ghactions-test:~$ minikube start --nodes 3
```

- Confirm it's running

```shell
ubuntu@kubectl-ghactions-test:~$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

minikube-m02
type: Worker
host: Running
kubelet: Running

minikube-m03
type: Worker
host: Running
kubelet: Running
```

5. Disable anonymous API access

```shell
ubuntu@kubectl-ghactions-test:~$ minikube ssh
docker@minikube:~$ sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

- Add `--anonymous-auth=false` under `command`
- Save and exit, minikube will automatically restart the API server, you may have to wait a few seconds

- See that anonymous requests are blocked

```shell
ubuntu@kubectl-ghactions-test:~$ curl -k https://$(minikube ip):8443/api/v1/namespaces/default/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}

ubuntu@kubectl-ghactions-test:~$ curl --cert /home/ubuntu/.minikube/profiles/minikube/client.crt \
      --key /home/ubuntu/.minikube/profiles/minikube/client.key \
      -k https://$(minikube ip):8443/api/v1/namespaces/default/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "1080"
  },
  "items": []
}
```

7. Run the kube-API proxy on the server

- We're going to use an apache2 webserver to proxy requests to the server to the minikube kube-API

  1. Install `apache2`

  ```shell
  ubuntu@kubectl-ghactions-test:~$ sudo apt install apache2
  ubuntu@kubectl-ghactions-test:~$ sudo systemctl enable --now apache2
  ```

  - Note you can do this in a container if you want (podman/docker) but for simplicity's sake, I'm not.

  2. Add virtualhost to proxy requests to the kubctl proxy (words)

  ```
  # In /etc/apache2/sites-available/kubectl.conf

  <VirtualHost *:80>
          ServerName <domain or IP>

          ProxyPass / https://<minikube-ip>:8443/
          ProxyPassReverse / https://<minikube-ip>:8443/

          # Allow minikube's self signed certs
          SSLProxyEngine on
          SSLProxyVerify none
          SSLProxyCheckPeerCN off
          SSLProxyCheckPeerName off

          ErrorLog ${APACHE_LOG_DIR}/kubectl-error.log
          CustomLog ${APACHE_LOG_DIR}/kubectl-access.log combined
  </VirtualHost>
  ```

  3. Enable modules and start `apache2`

  ```shell
  ubuntu@kubectl-ghactions-test:~$ sudo a2enmod proxy
  ubuntu@kubectl-ghactions-test:~$ sudo a2enmod proxy_http
  ubuntu@kubectl-ghactions-test:~$ sudo a2enmod ssl
  ubuntu@kubectl-ghactions-test:~$ sudo systemctl enable --now apache2
  ubuntu@kubectl-ghactions-test:~$ sudo a2ensite kubectl
  ```

8. Check if the API is accessible remotely

```shell
$ curl -X GET <your-server-url>/api
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

9. Authorize with the API

- See [Using RBAC Authorization | Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

- On the minikube host:
  1. Create a service account, `ClusterRoleBinding`, and token

  ```shell
  ubuntu@kubectl-ghactions-test:~$ kubectl create serviceaccount remote-dev
  ubuntu@kubectl-ghactions-test:~$ kubectl create clusterrolebinding remote-dev-binding \
                                    --clusterrole=cluster-admin \
                                    --serviceaccount=default:remote-dev
  ubuntu@kubectl-ghactions-test:~$ kubectl create token remote-dev
  ```

- On your local machine, create a  kubectl config to point to and authenticate with the remote server
- Put the below in a file named kube-config, replacing `<server-url>` and `<JWT>` with their appropriate values

```yaml
apiVersion: v1
kind: Config

clusters:
- name: minikube
  cluster:
    server: <server-url>

users:
- name: remote-user
  user:
    token: <JWT>

contexts:
- name: remote-context
  context:
    cluster: minikube
    user: remote-user

current-context: remote-context
```

- Test access by making a request

```shell
$ KUBECONFIG=$(pwd)/kube-config ./kubectl get ns
NAME              STATUS   AGE
default           Active   7m57s
kube-node-lease   Active   7m57s
kube-public       Active   7m57s
kube-system       Active   7m57s
```
