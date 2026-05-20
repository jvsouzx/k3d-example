# vaultwarden

Validates the community [`guerzon/vaultwarden`](https://github.com/guerzon/vaultwarden) Helm chart against the same OpenBao + ESO + CNPG stack used by [`../fullstack-template`](../fullstack-template). Kustomize ships only the namespace, CNPG `Cluster` and `ExternalSecret`; the workload comes from `helm install` with checked-in values.

## Layout

```
vaultwarden/
├── base
│   ├── cluster.yaml
│   ├── kustomization.yaml
│   └── namespace.yaml
└── overlays
    └── local
        ├── external-secret.yaml
        ├── kustomization.yaml
        └── values.yaml
```

Key wiring in [`overlays/local/values.yaml`](./overlays/local/values.yaml):

- `database.existingSecretKey: uri` pulls `DATABASE_URL` straight from the `vaultwarden-pg-app` Secret that CNPG auto-generates.
- `adminToken.existingSecret: vaultwarden-admin` pulls `ADMIN_TOKEN` from the Secret materialized by ESO.
- Traefik ingress with TLS (Secret `vaultwarden-tls`, created in §1), `/data` PVC on the default local-path StorageClass.

## Prerequisites

Cluster bootstrap (CNPG operator, OpenBao, ESO, `ClusterSecretStore`) must already be in place. See [../README.md](../README.md) §3–§4.

Add `vaultwarden.local` to `/etc/hosts` (already in [`clusters/local/local.yaml`](../../clusters/local/local.yaml)'s `hostAliases`, but the host needs it too):

```
127.0.0.1 vaultwarden.local
```

## 1. Local TLS via mkcert

Vaultwarden's web vault uses the Web Crypto API, which browsers only expose in secure contexts (`https://`, `localhost`, `127.0.0.1`). For `https://vaultwarden.local:8443` to work without browser warnings, the host needs a locally-trusted cert.

```sh
# One-time on the host: install mkcert + register its root CA in OS/NSS trust stores
sudo apt install mkcert libnss3-tools
mkcert -install

# Per cluster: issue the cert and load it as a TLS Secret in the vaultwarden namespace
mkcert vaultwarden.local
kubectl create namespace vaultwarden --dry-run=client -o yaml | kubectl apply -f -
kubectl -n vaultwarden create secret tls vaultwarden-tls \
  --cert=vaultwarden.local.pem \
  --key=vaultwarden.local-key.pem
```

`mkcert -install` only needs to run once per host. The cert/key are regenerated when you recreate the cluster: re-run `mkcert vaultwarden.local` and the `create secret tls` line.

## 2. Extend the OpenBao `eso-reader` policy

The policy created in [../README.md](../README.md) §4.3 only grants `secret/data/ft/*`. Add the Vaultwarden path:

```sh
kubectl -n openbao exec -ti openbao-0 -- sh
bao login <root_token>

bao policy write eso-reader - <<EOF
path "secret/data/ft/*"          { capabilities = ["read"] }
path "secret/data/vaultwarden/*" { capabilities = ["read"] }
EOF

exit
```

## 3. Seed the admin token in OpenBao

Vaultwarden's admin token can be a plain string or an Argon2 PHC hash (recommended). Generate the hash with the upstream image, since the OpenBao pod has neither `vaultwarden` nor `argon2`:

```sh
docker run --rm -it vaultwarden/server:1.36.0-alpine /vaultwarden hash --preset bitwarden
# paste a password when prompted; copy the resulting "$argon2id$..." line
```

Then store it in OpenBao (use single quotes so the shell doesn't expand `$`):

```sh
kubectl -n openbao exec -ti openbao-0 -- sh
bao login <root_token>

bao kv put secret/vaultwarden/admin \
  ADMIN_TOKEN='$argon2id$v=19$m=...$...'

exit
```

## 4. Apply the Kustomize layer (CNPG + ESO)

```sh
kubectl apply -k apps/vaultwarden/overlays/local/
kubectl -n vaultwarden wait --for=condition=Ready cluster/vaultwarden-pg --timeout=300s
kubectl -n vaultwarden get externalsecret vaultwarden-admin  # STATUS should be SecretSynced
```

The CNPG cluster generates the `vaultwarden-pg-app` secret with the `uri` key the chart consumes. `helm install` will fail if the CNPG cluster isn't Ready yet (pods crash-loop on a missing secret).

## 5. Install the chart

```sh
helm repo add vaultwarden https://guerzon.github.io/vaultwarden
helm repo update
helm install vaultwarden vaultwarden/vaultwarden \
  -n vaultwarden \
  --version 0.36.4 \
  -f apps/vaultwarden/overlays/local/values.yaml
```

Verify:

```sh
kubectl -n vaultwarden get pods,svc,ingress,pvc
kubectl -n vaultwarden logs deploy/vaultwarden -f      # look for "Rocket has launched from http://[::]:8080"
```

Open <https://vaultwarden.local:8443>. The admin panel lives at `/admin` and prompts for the token you hashed in §3 (use the **plaintext** you typed into `vaultwarden hash`, not the PHC string).

## 6. Tear down

```sh
helm uninstall vaultwarden -n vaultwarden
kubectl delete -k apps/vaultwarden/overlays/local/   # removes CNPG cluster + namespace + ESO
```
