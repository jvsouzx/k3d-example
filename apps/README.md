# apps

Deploy of the [FastAPI Fullstack Template](https://github.com/fastapi/full-stack-fastapi-template) on a local k3d cluster, organized as Kustomize with `base/` + `overlays/<env>/`.

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

Secrets live in OpenBao and are synced into native `Secret`s by [External Secrets Operator](https://external-secrets.io). OpenBao's Helm values (Standalone, file storage, UI, Ingress at `bao.local`) live in [../clusters/local/bootstrap/openbao/values.yaml](../clusters/local/bootstrap/openbao/values.yaml).

### 4.1 Install ESO and OpenBao

```sh
# ESO
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace --version 0.20.4

# OpenBao
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update
helm install openbao openbao/openbao \
  -n openbao --create-namespace \
  -f clusters/local/bootstrap/openbao/values.yaml

# Grant OpenBao's SA permission to call TokenReview (required by Kubernetes auth)
kubectl apply -f clusters/local/bootstrap/openbao/auth-delegator.yaml
```

### 4.2 Initialize and unseal OpenBao

The `openbao-0` pod stays `0/1` Ready until you init and unseal:

```sh
# init — prints 5 unseal keys + the initial root token. SAVE THEM.
kubectl -n openbao exec -ti openbao-0 -- bao operator init

# unseal — repeat 3 times with 3 distinct keys
kubectl -n openbao exec -ti openbao-0 -- bao operator unseal
```

After the 3rd unseal the pod flips to `1/1` Ready and the UI is reachable at <http://bao.local:8080>.

> The file backend persists data in the PVC, but the unseal keys are **not** stored anywhere by OpenBao — losing them means losing the data. Back them up (password manager, KMS). For production, switch to auto-unseal with a cloud KMS.

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

If status stays `InvalidProviderConfig`, check `kubectl -n external-secrets logs deploy/external-secrets` — usually a wrong `server` URL, missing `auth-delegator` binding, or a mismatched role/policy.

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

> Shell substitution like `$(openssl ...)` runs **inside** the pod, where `openssl` is absent — it would silently produce an empty `SECRET_KEY`. Generate on the host, paste the literal.

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

If the `ExternalSecret` shows `SecretSyncedError`, check `kubectl -n ft describe externalsecret environment-credentials` — most failures are a missing key in OpenBao or a policy that doesn't grant read on the path.
