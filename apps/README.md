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

`base/` holds the generic resources (namespace, configmap, deployments, ingress, CNPG Cluster). `overlays/local/` extends the base and adds the `ExternalSecret`, which lives in the overlay because the Infisical project/environment it pulls from is environment-specific.

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
127.0.0.1 infisical.local
```

The k3d loadbalancer maps `8080:80`, so use:

- Frontend: <http://app.template.local:8080>
- API: <http://api.template.local:8080>
- Infisical UI: <http://infisical.local:8080>

## 3. Cluster bootstrap (once per cluster)

Installs the CloudNativePG and Dragonfly operators, defined in [../clusters/local/bootstrap/kustomization.yaml](../clusters/local/bootstrap/kustomization.yaml):

```sh
kubectl apply -k clusters/local/bootstrap/ --server-side
```

`--server-side` is required because the CNPG CRDs are large enough to exceed the 256 KB annotation limit of client-side apply.

## 4. Secrets management: Infisical + ESO

Secrets live in a self-hosted [Infisical](https://infisical.com) and are synced into native `Secret`s by [External Secrets Operator](https://external-secrets.io). The bundled Postgres/Redis are disabled — Postgres comes from a **CNPG `Cluster`** and Redis from a **Dragonfly** instance, matching the production stack. Infisical's Helm values live in [../clusters/local/bootstrap/infisical/values.yaml](../clusters/local/bootstrap/infisical/values.yaml).

### 4.1 Install ESO

```sh
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace --version 0.20.4
```

### 4.2 Provision Infisical's datastores (CNPG + Dragonfly)

```sh
kubectl create namespace infisical
kubectl apply -f clusters/local/bootstrap/infisical/postgres.yaml
kubectl apply -f clusters/local/bootstrap/infisical/dragonfly.yaml

# wait for both to be ready
kubectl -n infisical wait --for=condition=Ready cluster/infisical-postgres --timeout=300s
kubectl -n infisical rollout status statefulset/infisical-dragonfly
```

CNPG generates the `infisical-postgres-app` Secret (with a ready-made `uri`); Dragonfly exposes a Service `infisical-dragonfly:6379` with the password from [dragonfly.yaml](../clusters/local/bootstrap/infisical/dragonfly.yaml).

### 4.3 Create the Infisical platform Secret

Not committed — built per cluster, and `DB_CONNECTION_URI` depends on the password CNPG just generated:

```sh
DB_URI=$(kubectl -n infisical get secret infisical-postgres-app \
  -o jsonpath='{.data.uri}' | base64 -d)

kubectl -n infisical create secret generic infisical-secrets \
  --from-literal=AUTH_SECRET="$(openssl rand -base64 32)" \
  --from-literal=ENCRYPTION_KEY="$(openssl rand -hex 16)" \
  --from-literal=SITE_URL="http://infisical.local:8080" \
  --from-literal=DB_CONNECTION_URI="$DB_URI" \
  --from-literal=REDIS_URL="redis://:infisical@infisical-dragonfly.infisical.svc.cluster.local:6379"
```

### 4.4 Install Infisical

```sh
helm repo add infisical-helm-charts \
  https://dl.cloudsmith.io/public/infisical/helm-charts/helm/charts/
helm repo update
helm install infisical infisical-helm-charts/infisical-standalone \
  -n infisical \
  -f clusters/local/bootstrap/infisical/values.yaml
```

Wait until the Infisical backend pods are `Running` (a migration Job runs against the CNPG database first):

```sh
kubectl -n infisical get pods -w
```

### 4.5 First-run setup in the Infisical UI

Open <http://infisical.local:8080> and complete the initial admin account form (first signup becomes the instance admin — there is no default password).

Then, inside the app:

1. **Create a project** — name it so the slug is `fullstack-template` (must match `projectSlug` in the `ClusterSecretStore`). Verify the slug under Project Settings.
2. **Pick the `dev` environment** (created by default) — matches `environmentSlug`.
3. **Add the app secrets** at the root path `/`:
   - `SECRET_KEY` — value: run `openssl rand -hex 32` on your host and paste
   - `FIRST_SUPERUSER` — e.g. `admin@example.com`
   - `FIRST_SUPERUSER_PASSWORD` — e.g. `changeme`
   - `SMTP_USER` — empty
   - `SMTP_PASSWORD` — empty

### 4.6 Create a Machine Identity for ESO

ESO authenticates to Infisical with a Machine Identity using Universal Auth:

1. Organization Settings → **Access Control → Machine Identities → Create**. Auth method: **Universal Auth**.
2. Add it to the `fullstack-template` project with at least **read** access to the `dev` environment.
3. Copy the **Client ID** and **Client Secret**, then store them where the `ClusterSecretStore` expects them:

```sh
kubectl -n external-secrets create secret generic universal-auth-credentials \
  --from-literal=clientId="f127bb9b-2ed4-42b6-8d9c-33b3acb82deb" \
  --from-literal=clientSecret="84c5bf7c7f036e43ab16d99cc9902fb4bc41d6c7fdbf6fe1a63a6dd1bce1bfca"
```

### 4.7 Apply the `ClusterSecretStore`

```sh
kubectl apply -f clusters/local/bootstrap/infisical/cluster-secret-store.yaml
kubectl get clustersecretstore infisical   # STATUS should become "Valid"
```

If status stays `InvalidProviderConfig`, check `kubectl -n external-secrets logs deploy/external-secrets` — usually a wrong `hostAPI`, a slug mismatch (`projectSlug`/`environmentSlug`), or the Machine Identity lacking access to the project.

> `hostAPI` in the `ClusterSecretStore` points at the in-cluster Service `infisical-infisical-standalone-infisical.infisical.svc.cluster.local:8080`. Confirm the name with `kubectl -n infisical get svc` if ESO can't reach it.

## 5. Deploy the application

```sh
kubectl apply -k apps/fullstack-template/overlays/local/
```

Verify:

```sh
kubectl -n ft get pods,svc,ingress,externalsecret
kubectl -n ft get cluster.postgresql.cnpg.io ft-postgres
kubectl -n ft get secret environment-credentials   # materialized by ESO from Infisical
```

If the `ExternalSecret` shows `SecretSyncedError`, check `kubectl -n ft describe externalsecret environment-credentials` — most failures are a slug mismatch or the Machine Identity not having access to the `dev` environment.
