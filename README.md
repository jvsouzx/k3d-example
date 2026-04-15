# k3d example

Minimal [k3d](https://k3d.io) cluster config and setup notes.

> Requires Linux and Docker ≥ 20.0.0.

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
```

## Usage

```sh
k3d cluster create --config k3d-cluster/k3d-local-cluster.yaml
```

## References

- [k3d releases](https://k3d.io/stable/#releases) · [config file](https://k3d.io/stable/usage/configfile/#introduction)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) · [kubectx](https://github.com/ahmetb/kubectx) · [k9s](https://github.com/derailed/k9s)
