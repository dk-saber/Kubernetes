# Nginx-Webserver auf Kubernetes bereitstellen (Deployment + NodePort-Service)

🌍 [Français](README.fr.md) · [English](README.en.md)

## Kontext

Ein Entwicklungsteam erstellt eine statische Website und möchte sie auf einem Kubernetes-Cluster bereitstellen. Die Anwendung muss **hochverfügbar** und **skalierbar** sein. Das DevOps-Team entscheidet sich daher für ein Deployment mit mehreren Replikas, das über einen Service vom Typ NodePort außerhalb des Clusters erreichbar ist.

## Ziele

- Ein Deployment namens `nginx-deployment` erstellen:
  - Image `nginx:latest` (expliziter Tag);
  - Container mit dem Namen `nginx-container`;
  - `3` Replikas.
- Einen Service vom Typ `NodePort` namens `nginx-service` mit dem `nodePort` `30011` erstellen.

## Architektur

```
   Client ──▶ <Knoten-IP>:30011
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

## Voraussetzungen

- Ein funktionierender Kubernetes-Cluster.
- `kubectl` mit Zugriff auf den Cluster.

## Manifest

Datei `nginx.yaml`:

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

## Bereitstellung

```bash
kubectl apply -f nginx.yaml
```

## Überprüfung

```bash
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx-app
kubectl get svc nginx-service
kubectl get endpoints nginx-service
```

Erwartetes Ergebnis:

- Deployment mit `READY 3/3`;
- 3 Pods im Status `Running`;
- `NodePort`-Service mit `80:30011/TCP`;
- 3 IP-Adressen in den Endpoints.

Zugriffstest (optional):

```bash
curl http://<Knoten-IP>:30011
```

Die Startseite „Welcome to nginx!“ sollte angezeigt werden.

## Zentrale Konzepte

| Konzept | Aufgabe |
|---|---|
| **Deployment** | Beschreibt den Soll-Zustand (Anzahl Replikas, Image) und erstellt ausgefallene Pods automatisch neu. |
| **Service** | Stabiler Einstiegspunkt, der den Datenverkehr auf die Pods verteilt, deren IPs flüchtig sind. |
| **NodePort** | Öffnet auf jedem Knoten einen Port (Bereich 30000-32767) für den Zugriff von außerhalb des Clusters. |
| **Labels / Selector** | Verbindet den Service mit den Pods: Der Service leitet an Pods weiter, deren Labels zu seinem `selector` passen. |

## Beispiel und Gegenbeispiel

| ✅ Gutes Beispiel | ❌ Gegenbeispiel | Folge |
|---|---|---|
| `selector` des Service identisch mit dem Pod-Label (`app: nginx-app`) | Pods mit `app: nginx`, Service wählt `app: nginx-app` | Leere Endpoints: Der Service leitet keinen Traffic weiter |
| `image: nginx:latest` | `image: nginx` (ohne Tag) | Impliziter Tag, weniger eindeutig und nicht anforderungskonform |
| `nodePort: 30011` (im Bereich 30000-32767) | `nodePort: 80` | Von der API abgelehnt: außerhalb des erlaubten Bereichs |
| `selector.matchLabels` identisch mit `template.labels` | Unterschiedliche Werte | Das Deployment wird von der API abgelehnt |
| Ein Deployment verwaltet die Pods | 3 von Hand erstellte Pods | Kein automatischer Ersatz bei Ausfall, manuelle Updates |

> **Diagnose-Tipp**: Antwortet der Service nicht, beginne mit `kubectl get endpoints nginx-service`. Eine leere Liste deutet fast immer auf nicht übereinstimmende Labels hin.

## Aufräumen

```bash
kubectl delete -f nginx.yaml
```
