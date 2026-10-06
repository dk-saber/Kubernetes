# Deploying an Nginx Web Server on Kubernetes (Deployment + NodePort Service)

🌍 [Français](README.fr.md) · [Deutsch](README.de.md)

## Context

A development team is building a static website and wants to deploy it on a Kubernetes cluster. The application must be **highly available** and **scalable**. The DevOps team therefore decides to create a Deployment with multiple replicas, exposed outside the cluster through a NodePort Service.

## Objectives

- Create a Deployment named `nginx-deployment`:
  - image `nginx:latest` (explicit tag);
  - container named `nginx-container`;
  - `3` replicas.
- Create a `NodePort` Service named `nginx-service` with `nodePort` `30011`.

## Architecture

```
   Client ──▶ <node IP>:30011
                    │
          ┌─────────▼──────────┐
          │ Service            │
          │ nginx-service      │
          │ NodePort 30011→80  │
          └───┬──────┬──────┬──┘
              ▼      ▼      ▼
          ┌──────┐┌──────┐┌──────┐
          │ Pod 1││ Pod 2││ Pod 3│
          └──────┘└──────┘└──────┘
        Deployment: nginx-deployment
```

## Prerequisites

- A working Kubernetes cluster.
- `kubectl` configured to access the cluster.

## Manifest

File `nginx.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30011
```

## Deployment

```bash
kubectl apply -f nginx.yaml
```

## Verification

```bash
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx-app
kubectl get svc nginx-service
kubectl get endpoints nginx-service
```

Expected result:

- Deployment showing `READY 3/3`;
- 3 Pods in `Running` state;
- `NodePort` Service showing `80:30011/TCP`;
- 3 IP addresses listed in the endpoints.

Access test (optional):

```bash
curl http://<node-IP>:30011
```

The "Welcome to nginx!" home page should be displayed.

## Key Concepts

| Concept | Role |
|---|---|
| **Deployment** | Describes the desired state (replica count, image) and automatically recreates failed Pods. |
| **Service** | Stable entry point that load-balances traffic across Pods, whose IPs are ephemeral. |
| **NodePort** | Opens a port (range 30000-32767) on every node for access from outside the cluster. |
| **Labels / selector** | Mechanism linking the Service to the Pods: the Service routes to Pods whose labels match its `selector`. |

## Example and Counter-Example

| ✅ Good example | ❌ Counter-example | Consequence |
|---|---|---|
| Service `selector` identical to the Pod label (`app: nginx-app`) | Pods labeled `app: nginx`, Service selecting `app: nginx-app` | Empty endpoints: the Service routes no traffic |
| `image: nginx:latest` | `image: nginx` (no tag) | Implicit tag, less explicit and not compliant with the requirement |
| `nodePort: 30011` (within 30000-32767) | `nodePort: 80` | Rejected by the API: outside the allowed range |
| `selector.matchLabels` identical to `template.labels` | Different values | The Deployment is rejected by the API |
| A Deployment managing the Pods | 3 Pods created by hand | No automatic replacement on failure, manual updates |

> **Troubleshooting tip**: if the Service does not respond, start with `kubectl get endpoints nginx-service`. An empty list almost always means a label mismatch.

## Cleanup

```bash
kubectl delete -f nginx.yaml
```
