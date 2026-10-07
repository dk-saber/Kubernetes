# Umgebungsvariablen in einem Kubernetes-Pod ausgeben

🌍 [Français](README.fr.md) · [English](README.en.md)

## Kontext

Ein DevOps-Team bereitet die Voraussetzungen für eine Anwendung vor, die personalisierte Begrüßungen sendet. Bevor die eigentliche Anwendung bereitgestellt wird, prüft das Team an einem einfachen Pod, ob die Umgebungsvariablen korrekt in den Container injiziert werden und die erwarteten Werte enthalten.

Dieses README verwendet ein **generisches Beispiel**: Namen und Werte können durch eigene ersetzt werden.

## Ziele

- Einen Pod namens `env-demo` erstellen.
- Den Container `demo-container` nennen und das Image `bash` verwenden.
- Drei Umgebungsvariablen definieren:
  - `GREETING` = `Hello from`
  - `APP_NAME` = `Acme`
  - `DEPARTMENT` = `Engineering`
- Den Befehl `["/bin/sh", "-c", 'echo "$(GREETING) $(APP_NAME) $(DEPARTMENT)"']` verwenden.
- `restartPolicy: Never` setzen, um eine Neustart-Schleife (`CrashLoopBackOff`) zu vermeiden.

## Funktionsweise

```
 env:
 GREETING="Hello from"    ──┐
 APP_NAME="Acme"          ──┼──▶ Kubernetes ersetzt $(VAR)
 DEPARTMENT="Engineering" ──┘    vor dem Start des Containers
                                      │
                                      ▼
                     echo "Hello from Acme Engineering"
                                      │
                                      ▼
              kubectl logs ──▶ Hello from Acme Engineering
```

## Voraussetzungen

- Ein funktionierender Kubernetes-Cluster.
- `kubectl` mit Zugriff auf den Cluster.

## Manifest

Datei `env-demo.yaml`:

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

## Bereitstellung

```bash
kubectl apply -f env-demo.yaml
```

## Überprüfung

```bash
kubectl get pod env-demo
kubectl logs env-demo
```

Erwartetes Ergebnis:

```
Hello from Acme Engineering
```

Der Pod-Status ist `Completed`: Der Container hat seine Nachricht ausgegeben und sich normal beendet, was mit `restartPolicy: Never` gewollt ist.

> **Hinweis zu `-f`**: `kubectl logs -f` verfolgt die Logs live. Hier beendet sich der Container sofort, daher bringt `-f` gegenüber einem einfachen `kubectl logs` nichts.

## Zentrale Konzepte

| Konzept | Aufgabe |
|---|---|
| **Umgebungsvariable** | Name/Wert-Paar, das beim Start injiziert wird, um die Anwendung ohne Änderung des Images zu konfigurieren. |
| **`$(VAR)`** | Syntax von **Kubernetes** (nicht der Shell): wird vor dem Containerstart in `command` und `args` durch den Wert der Variable ersetzt. |
| **`restartPolicy: Never`** | Der Pod wird nach Beendigung des Containers nicht neu gestartet. Standardmäßig (`Always`) würde er in einer Schleife neu gestartet. |

## Beispiel und Gegenbeispiel

| ✅ Gutes Beispiel | ❌ Gegenbeispiel | Folge |
|---|---|---|
| `$(APP_NAME)` mit in `env` definierter Variable `APP_NAME` | `$(APP_NAME)` ohne die Variable in `env` | Der Text bleibt unverändert, die Shell versucht einen Befehl `APP_NAME` auszuführen: `not found` |
| `restartPolicy: Never` auf `spec`-Ebene des Pods | `restartPolicy` im Container platziert | Unbekanntes Feld, Manifest abgelehnt |
| Werte in Anführungszeichen (`"Hello from"`) | Werte ohne Anführungszeichen wie `yes` oder `123` | Risiko, dass YAML sie als Boolean oder Zahl interpretiert |
| Status `Completed` wird erwartet | Erwartung eines `Running`-Pods | Ein `Completed`-Pod ist bei einer einmaligen Aufgabe normal |
| YAML-Manifest für einen Befehl mit verschachtelten Anführungszeichen | `kubectl run` mit verschachtelten Anführungszeichen | Die lokale Shell interpretiert Zeichen vor dem Senden |

## Aufräumen

```bash
kubectl delete -f env-demo.yaml
```
