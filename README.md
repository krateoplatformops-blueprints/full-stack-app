# Full Stack App Blueprint

A Krateo blueprint that provisions a complete full-stack application environment on Kubernetes:

- **PostgreSQL** — via [CloudNativePG](https://cloudnative-pg.io/) (CNPG) operator
- **Redis** (optional) — via [OT-CONTAINER-KIT](https://github.com/OT-CONTAINER-KIT/redis-operator) operator; toggled with `app.redis.enabled`
- **Backend** — any containerised HTTP service; connects to the database and optionally to Redis
- **Frontend** — any containerised web server; proxies API requests to the backend

## Operator pattern (in a nutshell)

Kubernetes operators extend the cluster's API with custom resources and embed domain-specific operational knowledge in a controller. Instead of running imperative scripts to set up a database cluster or a cache, you **declare the desired state** in a custom resource (e.g. a `Cluster` for CNPG or a `Redis` for OT-CONTAINER-KIT) and the **operator's controller loop continuously reconciles the actual cluster state towards it**: creating Pods, Services, Secrets, and ConfigMaps as needed, and reacting to failures automatically.

In this blueprint, two operators are responsible for the stateful infrastructure:

- **CNPG** watches `Cluster` resources and provisions a high-availability PostgreSQL cluster, handles primary election, streaming replication, and secret rotation entirely on its own.
- **OT-CONTAINER-KIT Redis Operator** watches `Redis` resources and provisions the Redis instance with its backing storage.

The blueprint only needs to declare *what* is wanted; the operators handle *how* to get there.

```mermaid
flowchart LR
    CR["Custom Resource\n(Cluster / Redis)"]
    OP["Operator\ncontroller loop"]
    K8S["Kubernetes resources\n(Pods, Services, Secrets…)"]

    CR -- "declare desired state" --> OP
    OP -- "reconcile" --> K8S
    K8S -- "observe actual state" --> OP
```

> **Note** — In this blueprint the reconciled resources are Kubernetes-native objects (Pods, Services, Secrets). This is the common case, but the pattern is not limited to it: an operator can manage anything its controller can reach — cloud provider APIs, external databases, DNS records, SaaS services, and so on. The custom resource always lives on the cluster and represents the desired state: only what the controller reconciles against on the other end changes.

---

## Prerequisites

- **CNPG Operator** is already deployed with a standard Krateo installation.

- **Redis Operator (OT-CONTAINER-KIT)**:
```bash
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts
helm upgrade --install redis-operator ot-helm/redis-operator \
  --namespace ot-operators --create-namespace

kubectl wait deployment/redis-operator \
  --namespace ot-operators \
  --for=condition=Available \
  --timeout=120s
```

---

## Architecture

```mermaid
flowchart TD
    Browser["🌐 Browser"]

    subgraph k8s["Kubernetes"]
        Frontend["Frontend"]
        Backend["Backend"]

        subgraph cnpg["CloudNativePG Cluster (HA · 3 instances)"]
            PG_RW["PostgreSQL Primary\n&lt;clusterName&gt;-rw:5432"]
            PG_RO["PostgreSQL Replicas x2\n&lt;clusterName&gt;-ro:5432"]
            PG_RW -- replication --> PG_RO
        end

        Redis["Redis Cache\n&lt;release&gt;-redis:6379\n⚠️ no password - optional"]
    end

    Browser --> Frontend
    Frontend -- "proxy /api/*" --> Backend
    Backend --> cnpg
    Backend -- "if REDIS_ENABLED=true" --> Redis
```

### Authentication & connectivity details

**PostgreSQL (CNPG)**

The Helm chart contains a CNPG `Cluster` resource that provisions a PostgreSQL cluster with 1 primary and 2 replicas. The cluster is exposed internally via two Services: `{{ .Release.Name }}-pg-cluster-rw` for read-write operations (pointing to the primary) and `{{ .Release.Name }}-pg-cluster-ro` for read-only operations (pointing to the replicas).
CNPG then **automatically generates** a Kubernetes Secret named `{{ .Release.Name }}-pg-cluster-app` containing the credentials for that database owner. Those credentials are injected into the backend as environment variables:

| Env var | Source |
|---|---|
| `DB_HOST` | ConfigMap → `{{ .Release.Name }}-pg-cluster-rw.{{ .Release.Namespace }}.svc.cluster.local` |
| `DB_PORT` | ConfigMap → `5432` |
| `DB_NAME` | ConfigMap → Helm release name |
| `DB_USER` | Secret `{{ .Release.Name }}-pg-cluster-app` → key `username` |
| `DB_PASSWORD` | Secret `{{ .Release.Name }}-pg-cluster-app` → key `password` |

**Redis**

Redis is deployed via the OT-CONTAINER-KIT operator **without any password or TLS**. The instance is only reachable inside the cluster. The backend connects only when `REDIS_ENABLED=true` is set in the backend ConfigMap, which is rendered from Helm values. This allows the blueprint to support Redis as an optional component. 

| Env var | Value |
|---|---|
| `REDIS_ENABLED` | `"true"` / `"false"` (default `"false"`) |
| `REDIS_HOST` | `{{ .Release.Name }}-redis.{{ .Release.Namespace }}.svc.cluster.local` |
| `REDIS_PORT` | `6379` |

---

## Key values

| Value | Default | Description |
|---|---|---|
| `app.backend.image.repository` | — | Backend container image repository |
| `app.backend.image.tag` | `latest` | Backend container image tag |
| `app.backend.service.type` | `ClusterIP` | Backend service type |
| `app.backend.service.port` | `30085` | Backend NodePort (if service.type is NodePort) |
| `app.frontend.image.repository` | — | Frontend container image repository |
| `app.frontend.image.tag` | `latest` | Frontend container image tag |
| `app.frontend.service.type` | `ClusterIP` | Frontend service type |
| `app.frontend.service.port` | `30086` | Frontend NodePort (if service.type is NodePort) |
| `app.redis.enabled` | `false` | Enable Redis cache |
| `app.redis.image` | `quay.io/opstree/redis:v7.4.8` | Redis image (enum: v7.4.8 / v8.0.6 / v8.2.5) |
| `app.redis.storage` | `1Gi` | Redis PVC size (enum: 1Gi / 2Gi / 3Gi) |
| `app.cnpg.postgresVersion` | `18` | PostgreSQL major version (enum: 16 / 17 / 18) |
| `app.cnpg.instances` | `3` | Number of CNPG replicas |
| `app.cnpg.storage.size` | `1Gi` | CNPG PVC size per instance (enum: 1Gi / 3Gi / 5Gi) |
| `testing.loadTesting.enabled` | `true` | Deploy a load-test CronJob |
| `testing.loadTesting.schedule` | `*/5 * * * *` | CronJob schedule |
| `testing.loadTesting.scriptImage` | — | Load-test container image |

Full schema: [`blueprint/values.schema.json`](blueprint/values.schema.json)

---

## Install using Krateo Composable Operation

**1. Register the blueprint** by creating a `CompositionDefinition`:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: full-stack-app
  namespace: demo-system
spec:
  chart:
    repo: full-stack-app
    url: https://marketplace.krateo.io
    version: 1.0.1
EOF
```

Wait for it to be ready:

```bash
kubectl wait compositiondefinition/full-stack-app \
  --namespace demo-system \
  --for=condition=Ready \
  --timeout=120s
```

**2. Deploy an instance** by creating a `FullStackApp` resource. Use `metadata.name` as the Helm release name and `metadata.namespace` as the target namespace:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v1-0-1
kind: FullStackApp
metadata:
  name: fsa-1
  namespace: demo-system
spec:
  app:
    backend:
      image:
        repository: ghcr.io/krateoplatformops-blueprints/pixel-grid-backend
        tag: latest
      service:
        type: ClusterIP
    frontend:
      image:
        repository: ghcr.io/krateoplatformops-blueprints/pixel-grid-frontend
        tag: latest
      service:
        type: NodePort
        port: 30086
    redis:
      enabled: false
      image: quay.io/opstree/redis:v7.4.8
      storage: 1Gi
    cnpg:
      postgresVersion: "18"
      instances: 3
      storage:
        size: 1Gi
  testing:
    loadTesting:
      enabled: false
  portal:
    enabled: true
EOF
```

Wait for the composition to be ready:

```bash
kubectl wait fullstackapp/fsa-1 \
  --namespace demo-system \
  --for=condition=Ready \
  --timeout=300s
```
