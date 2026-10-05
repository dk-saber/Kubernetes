# Sidecar-Pattern in Kubernetes: nginx-Logs weiterleiten

🌍 [Français](README.fr.md) · [English](README.en.md)

## Kontext

Ein nginx-Webserver erzeugt Access- und Error-Logs. Diese Logs sind nicht kritisch genug, um ein persistentes Volume zu rechtfertigen, doch die Entwicklungsteams benötigen die letzten 24 Stunden, um Fehler und Störungen nachzuverfolgen. Die Logs müssen daher an einen Log-Aggregationsdienst weitergeleitet werden.

Nach dem Prinzip der **Trennung der Zuständigkeiten (Separation of Concerns)** liefert nginx ausschließlich Webseiten aus, während ein zweiter Container (der **Sidecar**) allein für die Weiterleitung der Logs zuständig ist. Beide Container laufen im selben Pod und tauschen Dateien über ein gemeinsames `emptyDir`-Volume aus.

## Ziele

- Einen Pod namens `webserver` erstellen.
- Ein `emptyDir`-Volume namens `shared-logs` definieren.
- Einen regulären Container `nginx-container` hinzufügen (Image `nginx:latest`).
- Einen Sidecar-Container `sidecar-container` hinzufügen (Image `ubuntu:latest`), der Folgendes ausführt:
  `sh -c "while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done"`
- `shared-logs` in beiden Containern unter `/var/log/nginx` einhängen.
- Sicherstellen, dass alle Container den Status `Running` haben.

## Architektur

```
┌──────────────────── Pod: webserver ─────────────────────┐
│                                                          │
│  ┌──────────────────┐            ┌───────────────────┐   │
│  │  nginx-container │ schreibt   │ sidecar-container │   │
│  │  (nginx:latest)  │──────┐     │  (ubuntu:latest)  │   │
│  └──────────────────┘      │ ┌──▶│  liest/leitet     │   │
│                            ▼ │   └───────────────────┘   │
│                  ┌──────────────────┐                    │
│                  │ emptyDir         │                    │
│                  │ shared-logs      │                    │
│                  │ /var/log/nginx   │                    │
│                  └──────────────────┘                    │
└──────────────────────────────────────────────────────────┘
```

## Voraussetzungen

- Ein Kubernetes-Cluster ab Version **1.28** (native Sidecars, ab 1.29 standardmäßig aktiviert).
- `kubectl` mit Zugriff auf den Cluster.

## Manifest

Datei `webserver.yaml`:

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
      restartPolicy: Always        # macht den Init-Container zum nativen Sidecar
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

## Bereitstellung

```bash
kubectl apply -f webserver.yaml
kubectl get pod webserver
```

Erwartetes Ergebnis:

```
NAME        READY   STATUS    RESTARTS   AGE
webserver   2/2     Running   0          10s
```

## Überprüfung

```bash
# Logs des Sidecars (ein Durchlauf alle 30 Sekunden)
kubectl logs webserver -c sidecar-container

# Inhalt des gemeinsamen Volumes
kubectl exec webserver -c sidecar-container -- ls -l /var/log/nginx

# Eine Access-Log-Zeile erzeugen (optional)
kubectl exec webserver -c nginx-container -- curl -s localhost -o /dev/null
```

> **Hinweis**: Im allerersten Durchlauf kann der Sidecar `No such file or directory` ausgeben, da er vor nginx startet. Das ist unkritisch: Die Dateien entstehen, sobald nginx läuft, und werden im nächsten Durchlauf gelesen.

## Zentrale Konzepte

| Konzept | Aufgabe |
|---|---|
| **Sidecar** | Hilfscontainer, der im selben Pod neben dem Hauptcontainer läuft. |
| **emptyDir** | Temporäres Volume, das mit dem Pod erstellt und mit ihm gelöscht wird. Ideal für unkritische Daten. |
| **Nativer Sidecar** | Init-Container mit `restartPolicy: Always`: Er startet zuerst, läuft aber während der gesamten Lebensdauer des Pods weiter. |

## Beispiel und Gegenbeispiel

| ✅ Gutes Beispiel | ❌ Gegenbeispiel |
|---|---|
| Init-Container mit `restartPolicy: Always`: Der Pod erreicht `2/2 Running`. | Klassischer Init-Container mit Endlosschleife ohne `restartPolicy`: Er beendet sich nie, der Pod hängt in `Init:0/1` und nginx startet nicht. |
| Ein Container pro Zuständigkeit (ausliefern / weiterleiten). | Ein einziger Container für nginx und Log-Versand: Ein Absturz des Skripts kann den Webserver beeinträchtigen. |
| Volume in beiden Containern unter demselben `mountPath` eingehängt. | Unterschiedliche Mount-Pfade: Der Sidecar sieht die von nginx geschriebenen Dateien nie. |

## Aufräumen

```bash
kubectl delete pod webserver
```
