# Printing Environment Variables in a Kubernetes Pod

🌍 [Français](README.fr.md) · [Deutsch](README.de.md)

## Context

A DevOps team is setting up the prerequisites for an application that will send personalized greetings. Before deploying the real application, the team tests on a simple Pod that environment variables are correctly injected into the container and carry the expected values.

This README uses a **generic example**: names and values can be replaced with your own.

## Objectives

- Create a Pod named `env-demo`.
- Name the container `demo-container` and use the `bash` image.
- Define three environment variables:
  - `GREETING` = `Hello from`
  - `APP_NAME` = `Acme`
  - `DEPARTMENT` = `Engineering`
- Use the command `["/bin/sh", "-c", 'echo "$(GREETING) $(APP_NAME) $(DEPARTMENT)"']`.
- Set `restartPolicy: Never` to avoid a restart loop (`CrashLoopBackOff`).

## How It Works

```
 env:
 GREETING="Hello from"    ──┐
 APP_NAME="Acme"          ──┼──▶ Kubernetes substitutes $(VAR)
 DEPARTMENT="Engineering" ──┘    before starting the container
                                      │
                                      ▼
                     echo "Hello from Acme Engineering"
                                      │
                                      ▼
              kubectl logs ──▶ Hello from Acme Engineering
```

## Prerequisites

- A working Kubernetes cluster.
- `kubectl` configured to access the cluster.

## Manifest

File `env-demo.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  restartPolicy: Never
  containers:
    - name: demo-container
      image: bash:latest
      command: ["/bin/sh", "-c", 'echo "$(GREETING) $(APP_NAME) $(DEPARTMENT)"']
      env:
        - name: GREETING
          value: "Hello from"
        - name: APP_NAME
          value: "Acme"
        - name: DEPARTMENT
          value: "Engineering"
```

## Deployment

```bash
kubectl apply -f env-demo.yaml
```

## Verification

```bash
kubectl get pod env-demo
kubectl logs env-demo
```

Expected result:

```
Hello from Acme Engineering
```

The Pod status is `Completed`: the container printed its message and exited normally, which is the intended behavior with `restartPolicy: Never`.

> **Note on `-f`**: `kubectl logs -f` follows logs live. Here the container exits immediately, so `-f` adds nothing compared with a plain `kubectl logs`.

## Key Concepts

| Concept | Role |
|---|---|
| **Environment variable** | Name/value pair injected at startup to configure the application without changing the image. |
| **`$(VAR)`** | **Kubernetes** syntax (not shell): replaced with the variable's value before the container starts, in `command` and `args`. |
| **`restartPolicy: Never`** | The Pod is not restarted once the container has finished. By default (`Always`) it would be restarted in a loop. |

## Example and Counter-Example

| ✅ Good example | ❌ Counter-example | Consequence |
|---|---|---|
| `$(APP_NAME)` with `APP_NAME` defined in `env` | `$(APP_NAME)` without the variable in `env` | The text stays untouched and the shell tries to run an `APP_NAME` command: `not found` |
| `restartPolicy: Never` at the Pod `spec` level | `restartPolicy` placed inside the container | Unknown field, manifest rejected |
| Quoted values (`"Hello from"`) | Unquoted values such as `yes` or `123` | Risk of YAML parsing them as a boolean or number |
| `Completed` status expected | Expecting a `Running` Pod | A `Completed` Pod is normal for a one-off task |
| YAML manifest for a command with nested quotes | `kubectl run` with nested quotes | The local shell interprets characters before sending |

## Cleanup

```bash
kubectl delete -f env-demo.yaml
```
