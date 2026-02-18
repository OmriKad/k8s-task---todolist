# Kubernetes Final Project - Helm Chart

This is an original Helm chart implementation for a full todolist stack:
- `webui` Deployment (frontend)
- `todo-api` Deployment (REST API)
- `backend` StatefulSet (MariaDB)

## Project Tree

```text
.
├── README.md
└── todolist/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── _helpers.tpl
        ├── configmap.yaml
        ├── deployment-api.yaml
        ├── deployment-frontend.yaml
        ├── ingress.yaml
        ├── NOTES.txt
        ├── secret.yaml
        ├── service-api.yaml
        ├── service-backend-headless.yaml
        ├── service-frontend.yaml
        ├── statefulset-backend.yaml
        └── tests/test-connection.yaml
```

## Required Rubric Coverage

1. Deployments for web UI and REST API:
   - `todolist/templates/deployment-frontend.yaml`
   - `todolist/templates/deployment-api.yaml`
2. Init containers:
   - API waits for DB
3. Probes:
   - readiness + liveness on frontend/API/DB
4. One StatefulSet (single replica) + headless service:
   - `todolist/templates/statefulset-backend.yaml`
   - `todolist/templates/service-backend-headless.yaml`
5. ConfigMap + Secret across components:
   - `todolist/templates/configmap.yaml`
   - `todolist/templates/secret.yaml`
6. `NOTES.txt` usage instructions:
   - `todolist/templates/NOTES.txt`
7. Ingress (`/` -> frontend, `/todos` -> API):
   - `todolist/templates/ingress.yaml`

## Required Configurable Parameters (values.yaml)

1. Number of replicas:
   - `frontend.replicaCount`
   - `api.replicaCount`
   - `mariadb.replicaCount`
2. Service type for frontend + API (default `LoadBalancer`):
   - `frontend.service.type`
   - `api.service.type`
3. Environment variables:
   - `env.user` (default: `kadmon-omri`, format: `lastname-firstname`)
   - `env.mysqlHost` (default: `backend`)
   - `secret.rootPassword` (no default, must be set at install)
   - `env.apiBaseUrl` (default: `/todos`)

## Exact Images Used

- frontend: `ghcr.io/bennyro-mta/todolist-vue:1.2`
- api: `ghcr.io/bennyro-mta/todos-api:1.2`
- mariadb: `mariadb:10`


## Install

Option1: OCI install (from published chart)

```bash
helm upgrade --install todolist-final \
  oci://ghcr.io/omrikad/k8s-task---todolist/todolist \
  --set secret.rootPassword='CHANGE_ME_STRONG_PASSWORD'
```

Option2: Local install from source (run from this repository root)

```bash
helm upgrade --install todolist-final ./todolist \
  --set secret.rootPassword='CHANGE_ME_STRONG_PASSWORD'
```

## One-Time Ingress Controller Install

If your cluster does not already have an ingress controller:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace
kubectl get ingressclass
```

## Verify

```bash
kubectl get deploy,sts,svc,ingress,pods
kubectl describe deploy webui
kubectl describe deploy todo-api
kubectl describe sts backend
kubectl describe ingress todolist-final-todolist
helm status todolist-final
```

Check env values from API pod:

```bash
kubectl exec deploy/todo-api -- printenv | grep -E 'USER|MYSQL_HOST|MY_SQL_ROOT_PASSWORD|API_BASE_URL'
```

Local testing (recommended for Docker Desktop/kind/minikube):

```bash
# macOS/Linux: add host mapping
echo "127.0.0.1 todolist.local" | sudo tee -a /etc/hosts

# forward ingress controller
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

Windows (PowerShell as Administrator): add `127.0.0.1 todolist.local` to `C:\Windows\System32\drivers\etc\hosts`.

Then open:
- `http://todolist.local:8080`
- `http://todolist.local:8080/todos`

Important:
- Ingress routes: `/` -> frontend, `/todos` -> todo-api.
- Frontend default is `API_BASE_URL=/todos`, so browser and API stay on the same host.

## Troubleshooting

1. UI shows `Network Error`
   - Confirm ingress route exists:
     - `kubectl get ingress`
   - Confirm ingress controller is running:
     - `kubectl get pods -n ingress-nginx`
   - Confirm ingress port-forward is running:
     - `kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80`
   - Confirm frontend runtime config:
     - `kubectl exec deploy/webui -- wget -qO- http://127.0.0.1:8080/config.js`
   - `API_BASE_URL` should stay `/todos` for ingress mode.

2. API pod logs show `Access denied` to MariaDB
   - With persistent volume, DB credentials are initialized on first run.
   - If you change `secret.rootPassword` later, API can fail auth.
   - Fix by either:
     - Reusing the original password used at first install, or
     - Resetting the project DB state (uninstall + delete PVC + reinstall).

## Learning Notes

1. `Deployment` is used for stateless frontend/API.
2. `StatefulSet` is used for database state + stable network identity.
3. Headless service (`clusterIP: None`) provides stable DNS for StatefulSet.
4. `ConfigMap` stores non-sensitive config.
5. `Secret` stores DB root password.
6. Init containers enforce startup dependency order.
7. Probes allow Kubernetes to route traffic only to healthy/ready pods.
