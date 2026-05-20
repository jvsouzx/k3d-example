# apps

Sample apps deployed to the local k3d cluster, organized as Kustomize with `base/` + `overlays/<env>/`:

- [`fullstack-template/`](./fullstack-template): [FastAPI Fullstack Template](https://github.com/fastapi/full-stack-fastapi-template), images built locally. Deploy flow below.
- [`vaultwarden/`](./vaultwarden): community Helm chart from [guerzon/vaultwarden-helm](https://github.com/guerzon/vaultwarden) backed by a dedicated CNPG cluster. Deploy flow in [vaultwarden/README.md](./vaultwarden/README.md).

## Layout

```
apps/
├── fullstack-template
│   ├── base
│   │   ├── backend-deployment.yaml
│   │   ├── cluster.yaml
│   │   ├── configmap.yaml
│   │   ├── frontend-deployment.yaml
│   │   ├── ingress.yaml
│   │   ├── kustomization.yaml
│   │   └── namespace.yaml
│   └── overlays
│       └── local
│           ├── external-secret.yaml
│           └── kustomization.yaml
├── vaultwarden            # see vaultwarden/README.md
└── README.md
```

`base/` holds the generic resources (namespace, configmap, deployments, ingress, CNPG Cluster). `overlays/local/` extends the base and adds the `ExternalSecret`, which lives in the overlay because the OpenBao path it pulls from is environment-specific.

## 1. Build the images

Clone the [FastAPI Fullstack Template](https://github.com/fastapi/full-stack-fastapi-template) and build the images locally.

### Frontend

`VITE_API_URL` is resolved at **build time** by Vite, so it must be correct when running `docker build`:

```sh
docker build -t ft-frontend:latest \
  --build-arg VITE_API_URL=http://api.template.local:8080 \
  -f frontend/Dockerfile .
```

### Backend

Before building, tweak the `CMD` in `backend/Dockerfile`. The upstream template uses `--workers 4`, but on Kubernetes you should scale via the Deployment's `replicas` and keep one worker per pod:

```diff
- CMD ["fastapi", "run", "--workers", "4", "app/main.py"]
+ CMD ["fastapi", "run", "app/main.py"]
```

```sh
docker build -t ft-backend:latest -f backend/Dockerfile .
```

### Import into k3d

The deployments use `imagePullPolicy: IfNotPresent` with tags that don't include a registry, so the images need to be available inside the cluster:

```sh
k3d image import ft-frontend:latest ft-backend:latest -c local
```

## 2. `/etc/hosts`

```
127.0.0.1 app.template.local
127.0.0.1 api.template.local
127.0.0.1 bao.local
```

The k3d loadbalancer maps `8080:80`, so use:

- Frontend: <http://app.template.local:8080>
- API: <http://api.template.local:8080>
- OpenBao UI: <http://bao.local:8080>

## 3. Cluster bootstrap (once per cluster)

Installs the CloudNativePG operator, defined in [../clusters/local/bootstrap/kustomization.yaml](../clusters/local/bootstrap/kustomization.yaml):

```sh
kubectl apply -k clusters/local/bootstrap/ --server-side
```

`--server-side` is required because the CNPG CRDs are large enough to exceed the 256 KB annotation limit of client-side apply.

## 4. Secrets management: OpenBao + ESO

Secrets live in OpenBao and are synced into native `Secret`s by [External Secrets Operator](https://external-secrets.io). OpenBao runs as a **single-replica StatefulSet with the PostgreSQL storage backend** ([docs](https://openbao.org/docs/configuration/storage/postgresql/)), backed by a dedicated CNPG cluster (`openbao-pg`, kept separate from the app's `ft-postgres`). All state lives in Postgres (no PVC, no HA, no Raft). Standalone is intentional for local testing; production would flip to HA. Helm values (`storage "postgresql"`, static-seal auto-unseal, UI, Ingress at `bao.local`) live in [../clusters/local/bootstrap/openbao/values.yaml](../clusters/local/bootstrap/openbao/values.yaml); the CNPG cluster in [../clusters/local/bootstrap/openbao/postgres-cluster.yaml](../clusters/local/bootstrap/openbao/postgres-cluster.yaml).

### 4.1 Install ESO and OpenBao

```sh
# ESO
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace --version 0.20.4

# OpenBao's PostgreSQL backend (namespace + dedicated CNPG cluster).
# Must be Ready BEFORE helm install, or the OpenBao pod crash-loops
# until the backend is reachable.
kubectl apply -f clusters/local/bootstrap/openbao/postgres-cluster.yaml
kubectl -n openbao wait --for=condition=Ready cluster/openbao-pg --timeout=300s

# Static-key auto-unseal: 32-byte AES-256 key as a K8s Secret. Generated
# on the host (kept out of git) and consumed via OPENBAO_UNSEAL_KEY.
# SAVE this key. Without it the vault can never be unsealed again.
kubectl -n openbao create secret generic openbao-unseal \
  --from-literal=key="$(openssl rand -base64 32)"

# OpenBao (namespace already created by the manifest above)
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update
helm install openbao openbao/openbao \
  -n openbao \
  -f clusters/local/bootstrap/openbao/values.yaml

# Grant OpenBao's SA permission to call TokenReview (required by Kubernetes auth)
kubectl apply -f clusters/local/bootstrap/openbao/auth-delegator.yaml
```

CNPG creates the `openbao-pg-app` secret (owner `openbao`, database `openbao`); the chart injects its `uri` key as `BAO_PG_CONNECTION_URL`. The `openbao_kv_store` table is auto-created on first start since the `openbao` role owns the database. CNPG serves TLS, so the connection negotiates SSL by default; for strict verification switch the URL to `sslmode=verify-full` and mount the `openbao-pg-ca` secret ([CNPG TLS](https://cloudnative-pg.io/documentation/current/ssl_connections/)).

### 4.2 Initialize OpenBao (auto-unseal)

With the `seal "static"` stanza, OpenBao **unseals itself on startup** from the `openbao-unseal` key, so there is no manual `bao operator unseal`. Run init once on the single replica:

```sh
# init: run ONCE on openbao-0. With auto-unseal this prints RECOVERY keys
# (for rekey/generate-root), not unseal keys, plus the root token. SAVE THEM.
kubectl -n openbao exec -ti openbao-0 -- bao operator init

kubectl -n openbao rollout status statefulset/openbao
```

Init unseals the vault immediately; subsequent pod restarts auto-unseal from the same key. The UI is reachable at <http://bao.local:8080>.

> Two independent secrets-of-last-resort now exist: the **`openbao-unseal` AES key** (without it the pod can never unseal; it is the root of trust) and the **recovery keys** from init (needed for rekey / `generate-root`). Neither is stored by OpenBao itself. Back both up (password manager, KMS). The static seal trades the manual Shamir ceremony for trusting whoever can read the `openbao-unseal` Secret. Acceptable for this local test, but for production use a real KMS/Transit seal instead ([static seal security note](https://openbao.org/docs/configuration/seal/static/)).

> **Re-initializing the vault.** To wipe it, recreate the Postgres backend: `helm uninstall openbao -n openbao`, then `kubectl delete -f clusters/local/bootstrap/openbao/postgres-cluster.yaml` and re-apply, then redo §4.1–4.2.

### 4.3 Configure OpenBao (KV engine, Kubernetes auth, ESO role)

Open a shell inside the pod and log in with the root token:

```sh
kubectl -n openbao exec -ti openbao-0 -- sh
bao login <root_token>
```

Then, inside the pod:

```sh
# KV v2 engine at path "secret/"
bao secrets enable -version=2 -path=secret kv

# Kubernetes auth (uses the pod's SA token to call TokenReview)
bao auth enable kubernetes
bao write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc.cluster.local"

# Policy: read-only access to secret/data/ft/*
bao policy write eso-reader - <<EOF
path "secret/data/ft/*" {
  capabilities = ["read"]
}
EOF

# Role: bind ESO's ServiceAccount to the eso-reader policy
bao write auth/kubernetes/role/eso \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-reader \
  ttl=1h

exit
```

### 4.4 Apply the `ClusterSecretStore`

```sh
kubectl apply -f clusters/local/bootstrap/openbao/cluster-secret-store.yaml
kubectl get clustersecretstore openbao   # STATUS should become "Valid"
```

If status stays `InvalidProviderConfig`, check `kubectl -n external-secrets logs deploy/external-secrets`. Usually a wrong `server` URL, missing `auth-delegator` binding, or a mismatched role/policy.

### 4.5 Seed the app secrets

The app expects 5 keys at `secret/ft/app` (consumed via `envFrom: secretRef: environment-credentials`).

Generate the `SECRET_KEY` **on the host** (the OpenBao image has no `openssl`):

```sh
SECRET_KEY=$(openssl rand -hex 32)
echo "$SECRET_KEY"   # copy this
```

Then, inside the pod:

```sh
kubectl -n openbao exec -ti openbao-0 -- sh
bao login <root_token>

bao kv put secret/ft/app \
  SECRET_KEY="<paste_here>" \
  FIRST_SUPERUSER="admin@example.com" \
  FIRST_SUPERUSER_PASSWORD="changeme" \
  SMTP_USER="" \
  SMTP_PASSWORD=""

exit
```

> Shell substitution like `$(openssl ...)` runs **inside** the pod, where `openssl` is absent, which would silently produce an empty `SECRET_KEY`. Generate on the host, paste the literal.

To update later, repeat `bao kv put` (replaces all keys) or use `bao kv patch` (merges).

## 5. Deploy the application

```sh
kubectl apply -k apps/fullstack-template/overlays/local/
```

Verify:

```sh
kubectl -n ft get pods,svc,ingress,externalsecret
kubectl -n ft get cluster.postgresql.cnpg.io ft-postgres
kubectl -n ft get secret environment-credentials   # materialized by ESO from OpenBao
```

If the `ExternalSecret` shows `SecretSyncedError`, check `kubectl -n ft describe externalsecret environment-credentials`. Most failures are a missing key in OpenBao or a policy that doesn't grant read on the path.
