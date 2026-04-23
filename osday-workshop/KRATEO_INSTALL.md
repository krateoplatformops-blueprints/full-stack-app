# Krateo Installation Guide

## Prerequisites

- docker
- kubectl
- helm v3
- kind (for local cluster setup)

## Add the Krateo Helm repository

```sh
helm repo add krateo https://charts.krateo.io
helm repo update krateo
```

## Install `krateoctl` 

Follow the instructions at: [https://github.com/krateoplatformops/krateoctl](https://github.com/krateoplatformops/krateoctl)

Verify the installation by running:
```sh
krateoctl
```

## Kind cluster Setup

Create a Kubernetes cluster using Kind with the necessary configuration for Krateo:
```sh
kind create cluster --wait 120s --image kindest/node:v1.33.4 --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: krateo-quickstart
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
  extraPortMappings:
  - containerPort: 30080 # Krateo Portal
    hostPort: 30080
  - containerPort: 30081 # Krateo Snowplow
    hostPort: 30081
  - containerPort: 30082 # Krateo AuthN Service
    hostPort: 30082
  - containerPort: 30083 # Krateo events-presenter
    hostPort: 30083
  - containerPort: 30085
    hostPort: 30085
  - containerPort: 30086
    hostPort: 30086
networking:
  # By default the API server listens on a random open port.
  # You may choose a specific port but probably don't need to in most cases.
  # Using a random port makes it easier to spin up multiple clusters.
  apiServerPort: 6443
EOF
```

## Verify that the cluster is up and running:
```sh
kubectl cluster-info --context kind-krateo-quickstart
```

## Krateo Installation

Create the `krateo-system` namespace and install Krateo using `krateoctl`:
```sh
kubectl create namespace krateo-system
krateoctl install apply --type nodeport --namespace krateo-system --init-secrets --version 3.0.0-rc2-dev13 --profile no-finops,debug
```

## Accessing the Krateo Portal

Get the admin password for the admin user:
```sh
kubectl get secret admin-password  -n krateo-system -o jsonpath="{.data.password}" | base64 -d
```

Access the Krateo Portal at: [http://localhost:30080](http://localhost:30080) using the username `admin` and the password retrieved in the previous step.

## Install the Krateo Portal Admin Page

```sh
helm install portal-admin-page portal-admin-page \
  --repo https://marketplace.krateo.io \
  --namespace krateo-system \
  --set portalBlueprintPage.installCompositionDefinition=false \
  --version 1.1.1 \
  --wait
```
