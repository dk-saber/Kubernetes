# Rolling Update and Rollback of a Kubernetes Deployment

🌍 [Français](README.fr.md) · [Deutsch](README.de.md)

## Context

A production release is planned for the following week. Before it happens, the DevOps team wants to rehearse a Deployment **update** and **rollback** on a development environment in order to identify the risks in advance.

This README uses a **generic example**: names, namespace and port can be replaced with your own.

## Objectives

- Create a namespace `demo`.
- Create a Deployment `web-deploy` in that namespace:
  - one container named `web`, image `httpd:2.4.28`, `3` replicas;
  - `RollingUpdate` strategy with `maxSurge: 1` and `maxUnavailable: 2`.
- Create a `NodePort` Service named `web-service` exposing the Deployment on `nodePort` `30080`.
- Update the Deployment to `httpd:2.4.43` using a rolling update.
- Once all Pods are updated, undo the update and roll back to the original version.

## How It Works

```
  ① 2.4.28             ② 2.4.43             ③ Rollback
┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│ RS A 2.4.28 ■■■│──▶│ RS A 2.4.28  — │──▶│ RS A 2.4.28 ■■■│
│ RS B  —        │   │ RS B 2.4.43 ■■■│   │ RS B 2.4.43  — │
└────────────────┘   └────────────────┘   └────────────────┘
RS = ReplicaSet · ■ = Pod
```

Each time the Pod template changes, the Deployment creates a new ReplicaSet and keeps the old one (with no Pods). A rollback simply reactivates the old ReplicaSet.

## Strategy Parameters (with 3 replicas)

| Parameter | Value | Meaning | Effect |
|---|---|---|---|
| `maxSurge` | 1 | Number of **extra** Pods allowed during the update | Up to 4 Pods at the same time |
| `maxUnavailable` | 2 | Number of Pods that **may be missing** | At least 1 Pod stays available |

## Prerequisites

- A working Kubernetes cluster.
- `kubectl` configured to access the cluster.

## Manifest

File `web.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
  namespace: demo
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: httpd:2.4.28
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: demo
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

## Procedure

**1. Initial deployment**

```bash
kubectl apply -f web.yaml
kubectl rollout status deployment/web-deploy -n demo
```

**2. Update to 2.4.43** (format `container-name=image`)

```bash
kubectl set image deployment/web-deploy web=httpd:2.4.43 -n demo
kubectl rollout status deployment/web-deploy -n demo
```

**3. Rollback** (once all Pods are updated)

```bash
kubectl rollout undo deployment/web-deploy -n demo
kubectl rollout status deployment/web-deploy -n demo
```

## Verification

```bash
# Image after the rollback (expected: httpd:2.4.28)
kubectl get deployment web-deploy -n demo -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

kubectl get pods -n demo
kubectl get svc -n demo
kubectl get rs -n demo
kubectl rollout history deployment/web-deploy -n demo

# Version actually served (expected: Apache/2.4.28)
curl -sI http://<node-IP>:30080 | grep -i server
```

Expected result: image `httpd:2.4.28`, 3 Pods `Running`, Service `80:30080/TCP`.

> **History**: after the rollback you will see revisions **2 and 3**, but no longer 1. The content of the old revision 1 is reclassified as the new revision 3. This is normal Kubernetes behavior.

## Key Concepts

| Concept | Role |
|---|---|
| **RollingUpdate** | Replaces Pods gradually, without a full service interruption. |
| **ReplicaSet** | Manages the Pods of a given version; a new one is created whenever the template changes. |
| **`rollout undo`** | Reactivates the ReplicaSet of the previous revision. |
| **NodePort** | Opens a port (range 30000-32767) on every node for external access. |

## Example and Counter-Example

| ✅ Good example | ❌ Counter-example | Consequence |
|---|---|---|
| Create the Deployment directly with the intended image | Create with another image, then fix it | Spurious revisions in the history |
| `-n demo` on every command | Forgetting `-n demo` | Resource not found, or created in `default` |
| `set image ... web=httpd:2.4.43` (container name) | `web-deploy=httpd:2.4.43` (Deployment name) | Error: container not found |
| Wait for `rollout status` before `undo` | Run `undo` in the middle of an update | Mix of old and new Pods during the rollback |
| `RollingUpdate` strategy | `Recreate` strategy | All Pods stop before new ones start: service outage |
| `nodePort: 30080` (range 30000-32767) | A port outside the range | Rejected by the API |

> **Warning**: `kubectl set image` does not modify the YAML file. Re-applying `kubectl apply -f web.yaml` afterwards puts the image back to `2.4.28` and creates an additional revision.

## Cleanup

```bash
kubectl delete -f web.yaml
```
