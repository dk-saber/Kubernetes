# Deploying Jenkins on Kubernetes

🌍 [Français](README.fr.md) · [Deutsch](README.de.md)

## Context

A DevOps team wants to set up a **Jenkins** continuous integration (CI) server to create and manage the deployment pipelines of its projects. Rather than installing it on a dedicated machine, the team hosts it on its Kubernetes cluster, which gives a reproducible setup (a single YAML file) and automatic restart on failure.

This README uses a **generic example**: names, namespace and port can be replaced with your own.

## Objectives

- Create a namespace `ci-tools`.
- Create a Deployment `jenkins` in that namespace:
  - label `app: jenkins`, `1` replica;
  - container named `jenkins`, image `jenkins/jenkins:lts`, port `8080`;
  - environment variable `JAVA_OPTS` = `-Djenkins.install.runSetupWizard=false` to skip the initial setup wizard.
- Create a `NodePort` Service named `jenkins-svc` exposing Jenkins on `nodePort` `30090`.
- Wait for the Pod to be `Running`, then check access to the UI in the browser.

## Architecture

```
 Browser ──▶ <node IP>:30090
                    │
         ┌──────────▼───────────┐
         │ Service jenkins-svc  │
         │ NodePort 30090 → 8080│
         └──────────┬───────────┘
                    ▼
         ┌──────────────────────┐
         │ Pod (app: jenkins)   │
         │ JAVA_OPTS=...        │
         └──────────────────────┘
   Deployment: jenkins (1 replica) · Namespace: ci-tools
```

## Prerequisites

- A working Kubernetes cluster.
- `kubectl` configured to access the cluster.

## Manifest

File `jenkins.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ci-tools
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-svc
  namespace: ci-tools
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30090
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: ci-tools
  labels:
    app: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          ports:
            - containerPort: 8080
          env:
            - name: JAVA_OPTS
              value: "-Djenkins.install.runSetupWizard=false"
```

## Deployment

```bash
kubectl apply -f jenkins.yaml
kubectl get pods -n ci-tools -w
```

Wait until the Pod is `1/1 Running` (`Ctrl+C` to leave the `-w` mode).

## Verification

```bash
kubectl get all -n ci-tools
kubectl logs -n ci-tools deployment/jenkins | tail -5
kubectl get nodes -o wide
curl -I http://<node-IP>:30090
```

Expected result:

- Pod `1/1 Running`, Service `8080:30090/TCP`;
- a message such as `Jenkins is fully up and running` in the logs;
- the Jenkins dashboard in the browser at `http://<node-IP>:30090`.

> **Be patient**: a `Running` Pod does not mean Jenkins is ready. At startup it loads its plugins for 1 to 2 minutes and shows "Please wait while Jenkins is getting ready to work". Wait for the page to reload by itself.

## Key Concepts

| Concept | Role |
|---|---|
| **Namespace** | Logical space isolating a project's resources from the rest of the cluster. |
| **Deployment** | Keeps the desired number of Jenkins Pods and recreates the Pod on failure. |
| **NodePort Service** | Stable entry point, reachable from outside through a port (range 30000-32767) opened on every node. |
| **`JAVA_OPTS`** | Passes options to the JVM; `runSetupWizard=false` disables the initial setup wizard. |

## Example and Counter-Example

| ✅ Good example | ❌ Counter-example | Consequence |
|---|---|---|
| `-n ci-tools` on every command | Forgetting `-n ci-tools` | "No resources found": you are looking at the `default` namespace |
| Service `selector` = Pod label (`app: jenkins`) | Different label on the Pods | Empty endpoints, no response in the browser |
| `targetPort: 8080` (Jenkins port) | `targetPort: 80` | Connection refused: Jenkins does not listen on 80 |
| Exact `JAVA_OPTS` option, quoted | Typo in the option | The "Unlock Jenkins" wizard still appears |
| Wait 1 to 2 minutes after `Running` | Testing the URL immediately | Temporary waiting page or 503 error |
| A `nodePort` free on the cluster | A NodePort already taken, even in another namespace | Rejected: NodePorts are unique **cluster-wide** |

## Limitations to Know

- **No authentication**: with `runSetupWizard=false`, Jenkins starts without security, so anyone with access to the URL can do anything. Acceptable for a test, never in production.
- **No persistent volume**: Jenkins data (`/var/jenkins_home`) is lost if the Pod is recreated. In production, a `PersistentVolumeClaim` is mounted.
- **Image version**: `jenkins/jenkins:lts` follows the latest LTS release. In production, pin a precise version for reproducible deployments.

## Cleanup

```bash
kubectl delete -f jenkins.yaml
```
