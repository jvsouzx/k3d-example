# k3d example

Minimal [k3d](https://k3d.io) cluster config plus a sample app deployed via Kustomize, with secrets managed by Infisical + External Secrets Operator.

> Requires Linux and Docker ≥ 20.0.0.

## Layout

```
.
├── clusters/local/         # k3d config + cluster bootstrap (CNPG, Infisical, ESO)
│   ├── bootstrap/
│   │   ├── kustomization.yaml
│   │   └── infisical/      # Infisical Helm values + ClusterSecretStore
│   └── local.yaml
└── apps/                   # applications deployed to the cluster
    └── fullstack-template/
        ├── base/
        └── overlays/local/
```

## Setup

```sh
# k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# kubectx
sudo apt install kubectx

# k9s
wget https://github.com/derailed/k9s/releases/latest/download/k9s_linux_amd64.deb \
  && sudo apt install ./k9s_linux_amd64.deb \
  && rm k9s_linux_amd64.deb

# helm (used to install Infisical and External Secrets Operator)
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" \
  | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update && sudo apt-get install helm
```

## Usage

```sh
k3d cluster create --config clusters/local/local.yaml
```

k3d merges the new context into `~/.kube/config` and switches to it. List or swap contexts with `kubectx`.

Once the cluster is up, see [apps/README.md](apps/README.md) for the bootstrap (CNPG operator, Infisical, External Secrets Operator) and the application deploy flow.

Delete the cluster when done:

```sh
k3d cluster delete local
```

### Multiple kubeconfigs

If `$KUBECONFIG` has multiple paths (e.g. `foo.yaml:~/.kube/config`), k3d can't decide which file to update and warns on create/delete. Prefix both commands to scope the write:

```sh
KUBECONFIG=~/.kube/config k3d cluster create --config clusters/local/local.yaml
KUBECONFIG=~/.kube/config k3d cluster delete local
```

Or add a shell alias in `~/.zshrc` / `~/.bashrc`:

```sh
alias k3d='KUBECONFIG=~/.kube/config k3d'
```

If stale entries got left behind from an un-prefixed delete, clean them manually:

```sh
KUBECONFIG=~/.kube/config kubectl config delete-context k3d-local
KUBECONFIG=~/.kube/config kubectl config delete-cluster k3d-local
KUBECONFIG=~/.kube/config kubectl config delete-user    admin@k3d-local
```

## References

- [k3d releases](https://k3d.io/stable/#releases) · [config file](https://k3d.io/stable/usage/configfile/#introduction)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) · [kubectx](https://github.com/ahmetb/kubectx) · [k9s](https://github.com/derailed/k9s)
