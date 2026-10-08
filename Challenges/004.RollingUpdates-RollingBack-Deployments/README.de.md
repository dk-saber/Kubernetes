# Rolling Update und Rollback eines Kubernetes-Deployments

🌍 [Français](README.fr.md) · [English](README.en.md)

## Kontext

Für die folgende Woche ist ein Produktiv-Release geplant. Zuvor möchte das DevOps-Team das **Update** und den **Rollback** eines Deployments in einer Entwicklungsumgebung durchspielen, um Risiken frühzeitig zu erkennen.

Dieses README verwendet ein **generisches Beispiel**: Namen, Namespace und Port können durch eigene ersetzt werden.

## Ziele

- Einen Namespace `demo` erstellen.
- Ein Deployment `web-deploy` in diesem Namespace erstellen:
  - ein Container namens `web`, Image `httpd:2.4.28`, `3` Replikas;
  - Strategie `RollingUpdate` mit `maxSurge: 1` und `maxUnavailable: 2`.
- Einen Service vom Typ `NodePort` namens `web-service` erstellen, der das Deployment über den `nodePort` `30080` bereitstellt.
- Das Deployment per Rolling Update auf `httpd:2.4.43` aktualisieren.
- Sobald alle Pods aktualisiert sind, das Update rückgängig machen und auf die ursprüngliche Version zurückkehren.

## Funktionsweise

```
  ① 2.4.28             ② 2.4.43             ③ Rollback
┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│ RS A 2.4.28 ■■■│──▶│ RS A 2.4.28  — │──▶│ RS A 2.4.28 ■■■│
│ RS B  —        │   │ RS B 2.4.43 ■■■│   │ RS B 2.4.43  — │
└────────────────┘   └────────────────┘   └────────────────┘
RS = ReplicaSet · ■ = Pod
```

Bei jeder Änderung des Pod-Templates erstellt das Deployment ein neues ReplicaSet und behält das alte (ohne Pods). Ein Rollback reaktiviert einfach das alte ReplicaSet.

## Strategie-Parameter (bei 3 Replikas)

| Parameter | Wert | Bedeutung | Wirkung |
|---|---|---|---|
| `maxSurge` | 1 | Anzahl **zusätzlicher** Pods während des Updates | Bis zu 4 Pods gleichzeitig |
| `maxUnavailable` | 2 | Anzahl Pods, die **fehlen dürfen** | Mindestens 1 Pod bleibt verfügbar |

## Voraussetzungen

- Ein funktionierender Kubernetes-Cluster.
- `kubectl` mit Zugriff auf den Cluster.

## Manifest

Datei `web.yaml`:

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

## Ablauf

**1. Erstbereitstellung**

```bash
kubectl apply -f web.yaml
kubectl rollout status deployment/web-deploy -n demo
```

**2. Update auf 2.4.43** (Format `Containername=Image`)

```bash
kubectl set image deployment/web-deploy web=httpd:2.4.43 -n demo
kubectl rollout status deployment/web-deploy -n demo
```

**3. Rollback** (sobald alle Pods aktualisiert sind)

```bash
kubectl rollout undo deployment/web-deploy -n demo
kubectl rollout status deployment/web-deploy -n demo
```

## Überprüfung

```bash
# Image nach dem Rollback (erwartet: httpd:2.4.28)
kubectl get deployment web-deploy -n demo -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

kubectl get pods -n demo
kubectl get svc -n demo
kubectl get rs -n demo
kubectl rollout history deployment/web-deploy -n demo

# Tatsächlich ausgelieferte Version (erwartet: Apache/2.4.28)
curl -sI http://<Knoten-IP>:30080 | grep -i server
```

Erwartetes Ergebnis: Image `httpd:2.4.28`, 3 Pods `Running`, Service `80:30080/TCP`.

> **Historie**: Nach dem Rollback siehst du die Revisionen **2 und 3**, aber nicht mehr die 1. Der Inhalt der alten Revision 1 wird als neue Revision 3 geführt. Das ist normales Kubernetes-Verhalten.

## Zentrale Konzepte

| Konzept | Aufgabe |
|---|---|
| **RollingUpdate** | Ersetzt Pods schrittweise, ohne vollständige Unterbrechung des Dienstes. |
| **ReplicaSet** | Verwaltet die Pods einer bestimmten Version; bei jeder Template-Änderung wird ein neues erstellt. |
| **`rollout undo`** | Reaktiviert das ReplicaSet der vorherigen Revision. |
| **NodePort** | Öffnet auf jedem Knoten einen Port (Bereich 30000-32767) für externen Zugriff. |

## Beispiel und Gegenbeispiel

| ✅ Gutes Beispiel | ❌ Gegenbeispiel | Folge |
|---|---|---|
| Deployment direkt mit dem gewünschten Image erstellen | Mit einem anderen Image erstellen und danach korrigieren | Überflüssige Revisionen in der Historie |
| `-n demo` bei jedem Befehl | `-n demo` vergessen | Ressource nicht gefunden oder in `default` erstellt |
| `set image ... web=httpd:2.4.43` (Containername) | `web-deploy=httpd:2.4.43` (Deployment-Name) | Fehler: Container nicht gefunden |
| Vor `undo` auf `rollout status` warten | `undo` mitten im Update ausführen | Mischung aus alten und neuen Pods während des Rollbacks |
| Strategie `RollingUpdate` | Strategie `Recreate` | Alle Pods werden vor den neuen gestoppt: Dienstausfall |
| `nodePort: 30080` (Bereich 30000-32767) | Ein Port außerhalb des Bereichs | Von der API abgelehnt |

> **Achtung**: `kubectl set image` ändert die YAML-Datei nicht. Ein erneutes `kubectl apply -f web.yaml` setzt das Image auf `2.4.28` zurück und erzeugt eine zusätzliche Revision.

## Aufräumen

```bash
kubectl delete -f web.yaml
```
