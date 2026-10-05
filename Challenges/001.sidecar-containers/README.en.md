# Sidecar Pattern on Kubernetes: Shipping nginx Logs

🌍 [Français](README.fr.md) · [Deutsch](README.de.md)

## Context

An nginx web server produces access and error logs. These logs are not critical enough to justify a persistent volume, but development teams need access to the last 24 hours to trace bugs and incidents. The logs therefore have to be shipped to a log-aggregation service.

Following the **separation of concerns** principle, nginx only serves web pages, while a second container (the **sidecar**) is dedicated to shipping the logs. Both containers live in the same Pod and exchange files through a shared `emptyDir` volume.

## Objectives

- Create a Pod named `webserver`.
- Declare an `emptyDir` volume named `shared-logs`.
- Add a regular container `nginx-container` (image `nginx:latest`).
- Add a sidecar container `sidecar-container` (image `ubuntu:latest`) running:
  `sh -c "while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done"`
- Mount `shared-logs` in both containers at `/var/log/nginx`.
- Make sure all containers are in the `Running` state.

## Architecture

```
┌──────────────────── Pod: webserver ─────────────────────┐
│                                                          │
│  ┌──────────────────┐            ┌───────────────────┐   │
│  │  nginx-container │  writes    │  sidecar-container│   │
│  │  (nginx:latest)  │──────┐     │  (ubuntu:latest)  │   │
│  └──────────────────┘      │ ┌──▶│  reads and ships  │   │
│                            ▼ │   └───────────────────┘   │
│                  ┌──────────────────┐                    │
│                  │ emptyDir         │                    │
│                  │ shared-logs      │                    │
│                  │ /var/log/nginx   │                    │
│                  └──────────────────┘                    │
└──────────────────────────────────────────────────────────┘
```

## Prerequisites

- A Kubernetes cluster running version **1.28 or higher** (native sidecars, enabled by default from 1.29).
- `kubectl` configured to access the cluster.

## Manifest

File `webserver.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  volumes:
    - name: shared-logs
      emptyDir: {}
  initContainers:
    - name: sidecar-container
      image: ubuntu:latest
      restartPolicy: Always        # turns the init container into a native sidecar
      command: ["sh","-c","while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
  containers:
    - name: nginx-container
      image: nginx:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
```

## Deployment

```bash
kubectl apply -f webserver.yaml
kubectl get pod webserver
```

Expected result:

```
NAME        READY   STATUS    RESTARTS   AGE
webserver   2/2     Running   0          10s
```

## Verification

```bash
# Sidecar logs (one iteration every 30 seconds)
kubectl logs webserver -c sidecar-container

# Content of the shared volume
kubectl exec webserver -c sidecar-container -- ls -l /var/log/nginx

# Generate an access log line (optional)
kubectl exec webserver -c nginx-container -- curl -s localhost -o /dev/null
```

> **Note**: during the very first iteration, the sidecar may print `No such file or directory` because it starts before nginx. This is not a problem: the files appear as soon as nginx is up and are read on the next iteration.

## Key Concepts

| Concept | Role |
|---|---|
| **Sidecar** | Helper container running alongside the main container in the same Pod. |
| **emptyDir** | Temporary volume created with the Pod and deleted with it. Ideal for non-critical data. |
| **Native sidecar** | Init container with `restartPolicy: Always`: it starts first but keeps running for the whole life of the Pod. |

## Example and Counter-Example

| ✅ Good example | ❌ Counter-example |
|---|---|
| Init container with `restartPolicy: Always`: the Pod reaches `2/2 Running`. | Classic init container with an infinite loop and no `restartPolicy`: it never terminates, the Pod stays stuck in `Init:0/1` and nginx never starts. |
| One container per responsibility (serve / ship). | A single container doing both nginx and log shipping: a script crash can affect the web server. |
| Volume mounted at the same `mountPath` in both containers. | Different mount paths: the sidecar never sees the files written by nginx. |

## Cleanup

```bash
kubectl delete pod webserver
```
