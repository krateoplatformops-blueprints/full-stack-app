# Full Stack App Blueprint

A Krateo blueprint that provisions a complete full-stack application environment on Kubernetes:

- **Namespace** — isolated per deployment
- **PostgreSQL** — via [CloudNativePG](https://cloudnative-pg.io/) (CNPG) operator
- **Redis** (optional) — via [OT-CONTAINER-KIT](https://github.com/OT-CONTAINER-KIT/redis-operator) operator; toggled with `app.redis.enabled`
- **Backend** — any containerised HTTP service; connects to the database and optionally to Redis
- **Frontend** — any containerised web server; proxies API requests to the backend
- **Load test** (optional) — periodic CronJob running [Vegeta](https://github.com/tsenart/vegeta) inside the cluster

---

## Prerequisites

Install both operators in the cluster **once**, before applying the blueprint.

**CNPG Operator**
```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system --create-namespace \
  --version 0.27.1
```

**Redis Operator (OT-CONTAINER-KIT)**
```bash
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts
helm upgrade --install redis-operator ot-helm/redis-operator \
  --namespace ot-operators --create-namespace
```

---

## Architecture

```
Browser
  │
  ├─ GET /         → Frontend (nginx, port 8080)
  │                       │
  └─ /api/*  ─────────────┤  proxy /api/ → Backend (port 8080)
                                                │            │
                                          PostgreSQL       Redis
                                          (CNPG, RW svc)  (optional)
```

CNPG automatically creates:
- a read-write service at `<clusterName>-rw.<namespace>.svc.cluster.local`
- a secret `<clusterName>-app` with keys `username` and `password`

The backend receives DB credentials from that secret and connection details from a ConfigMap.

---

## Key values

| Value | Default | Description |
|---|---|---|
| `app.name` | — | Name prefix for all Kubernetes resources |
| `app.namespace` | — | Target namespace |
| `app.backend.image.repository` | — | Backend container image repository |
| `app.backend.image.tag` | `latest` | Backend container image tag |
| `app.backend.service.type` | `NodePort` | Backend service type |
| `app.backend.service.port` | `30086` | Backend NodePort |
| `app.frontend.image.repository` | — | Frontend container image repository |
| `app.frontend.image.tag` | `latest` | Frontend container image tag |
| `app.frontend.service.type` | `NodePort` | Frontend service type |
| `app.frontend.service.port` | `30088` | Frontend NodePort |
| `app.redis.enabled` | `false` | Enable Redis sidecar cache |
| `app.redis.image` | `quay.io/opstree/redis:v7.4.8` | Redis image (enum) |
| `app.redis.storage` | `1Gi` | Redis PVC size (enum: 1Gi / 2Gi / 3Gi) |
| `app.cnpg.clusterName` | `pg-cluster` | CNPG Cluster resource name |
| `app.cnpg.postgresVersion` | `18` | PostgreSQL major version (enum: 16 / 17 / 18) |
| `app.cnpg.instances` | `3` | Number of CNPG replicas |
| `app.cnpg.storage.size` | `1Gi` | CNPG PVC size per instance (enum: 1Gi / 3Gi / 5Gi) |
| `testing.loadTesting.enabled` | `true` | Deploy the load-test CronJob |
| `testing.loadTesting.schedule` | `*/5 * * * *` | CronJob schedule |
| `testing.loadTesting.scriptImage` | — | Load-test container image |

Full schema: [`blueprint/values.schema.json`](blueprint/values.schema.json)

---

## Install using Krateo Composable Operation

Install the `CompositionDefinition` to register the blueprint:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: full-stack-app
  namespace: krateo-system
spec:
  chart:
    repo: full-stack-app
    url: https://marketplace.krateo.io
    version: <version>
EOF
```

Then install the blueprint by creating a `FullStackApp` resource. Use `metadata.name` as the Helm release name and `metadata.namespace` as the target namespace:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v<version-with-dashes>
kind: FullStackApp
metadata:
  name: <release-name>
  namespace: <release-namespace>
spec:
  app:
    name: <release-name>
    namespace: <release-namespace>
    backend:
      image:
        repository: ghcr.io/my-org/my-backend
        tag: latest
      service:
        type: NodePort
        port: 30086
    frontend:
      image:
        repository: ghcr.io/my-org/my-frontend
        tag: latest
      service:
        type: NodePort
        port: 30088
    redis:
      enabled: false
      image: quay.io/opstree/redis:v7.4.8
      storage: 1Gi
    cnpg:
      clusterName: pg-cluster
      postgresVersion: "18"
      instances: 3
      storage:
        size: 1Gi
  testing:
    loadTesting:
      enabled: true
      schedule: "*/5 * * * *"
      scriptImage: ghcr.io/my-org/my-load-test:latest
EOF
```

---

## Source code

Application source, Dockerfiles, and CI workflows live in a separate repository.
Images are built independently and referenced here by tag — the blueprint has no knowledge of how they are built.
