# Jenkins auf Kubernetes bereitstellen

🌍 [Français](README.fr.md) · [English](README.en.md)

## Kontext

Ein DevOps-Team möchte einen **Jenkins**-Server für Continuous Integration (CI) einrichten, um die Pipelines seiner Projekte zu erstellen und zu verwalten. Statt ihn auf einer dedizierten Maschine zu installieren, betreibt das Team ihn auf seinem Kubernetes-Cluster. So entsteht eine reproduzierbare Installation (eine YAML-Datei) mit automatischem Neustart bei Ausfällen.

Dieses README verwendet ein **generisches Beispiel**: Namen, Namespace und Port können durch eigene ersetzt werden.

## Ziele

- Einen Namespace `ci-tools` erstellen.
- Ein Deployment `jenkins` in diesem Namespace erstellen:
  - Label `app: jenkins`, `1` Replika;
  - Container namens `jenkins`, Image `jenkins/jenkins:lts`, Port `8080`;
  - Umgebungsvariable `JAVA_OPTS` = `-Djenkins.install.runSetupWizard=false`, um den Einrichtungsassistenten zu überspringen.
- Einen Service vom Typ `NodePort` namens `jenkins-svc` erstellen, der Jenkins über den `nodePort` `30090` bereitstellt.
- Warten, bis der Pod `Running` ist, und danach den Zugriff auf die Oberfläche im Browser prüfen.

## Architektur

```
 Browser ──▶ <Knoten-IP>:30090
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
   Deployment: jenkins (1 Replika) · Namespace: ci-tools
```

## Voraussetzungen

- Ein funktionierender Kubernetes-Cluster.
- `kubectl` mit Zugriff auf den Cluster.

## Manifest

Datei `jenkins.yaml`:

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

## Bereitstellung

```bash
kubectl apply -f jenkins.yaml
kubectl get pods -n ci-tools -w
```

Warte, bis der Pod `1/1 Running` ist (`Ctrl+C`, um den `-w`-Modus zu verlassen).

## Überprüfung

```bash
kubectl get all -n ci-tools
kubectl logs -n ci-tools deployment/jenkins | tail -5
kubectl get nodes -o wide
curl -I http://<Knoten-IP>:30090
```

Erwartetes Ergebnis:

- Pod `1/1 Running`, Service `8080:30090/TCP`;
- in den Logs eine Meldung wie `Jenkins is fully up and running`;
- im Browser das Jenkins-Dashboard unter `http://<Knoten-IP>:30090`.

> **Geduld**: Ein `Running`-Pod bedeutet nicht, dass Jenkins bereit ist. Beim Start lädt es 1 bis 2 Minuten lang seine Plugins und zeigt „Please wait while Jenkins is getting ready to work“. Warte, bis sich die Seite von selbst neu lädt.

## Zentrale Konzepte

| Konzept | Aufgabe |
|---|---|
| **Namespace** | Logischer Raum, der die Ressourcen eines Projekts vom Rest des Clusters trennt. |
| **Deployment** | Hält die gewünschte Anzahl an Jenkins-Pods aufrecht und erstellt den Pod bei Ausfall neu. |
| **NodePort-Service** | Stabiler Einstiegspunkt, von außen über einen auf jedem Knoten geöffneten Port (Bereich 30000-32767) erreichbar. |
| **`JAVA_OPTS`** | Übergibt Optionen an die JVM; `runSetupWizard=false` deaktiviert den Einrichtungsassistenten. |

## Beispiel und Gegenbeispiel

| ✅ Gutes Beispiel | ❌ Gegenbeispiel | Folge |
|---|---|---|
| `-n ci-tools` bei jedem Befehl | `-n ci-tools` vergessen | „No resources found“: Du siehst den Namespace `default` |
| `selector` des Service = Pod-Label (`app: jenkins`) | Anderes Label bei den Pods | Leere Endpoints, keine Antwort im Browser |
| `targetPort: 8080` (Jenkins-Port) | `targetPort: 80` | Verbindung abgelehnt: Jenkins lauscht nicht auf 80 |
| Exakte `JAVA_OPTS`-Option in Anführungszeichen | Tippfehler in der Option | Der Assistent „Unlock Jenkins“ erscheint trotzdem |
| 1 bis 2 Minuten nach `Running` warten | URL sofort testen | Vorübergehende Wartseite oder 503-Fehler |
| Ein im Cluster freier `nodePort` | Ein bereits belegter NodePort, auch in einem anderen Namespace | Abgelehnt: NodePorts sind **clusterweit** eindeutig |

## Zu beachtende Einschränkungen

- **Keine Authentifizierung**: Mit `runSetupWizard=false` startet Jenkins ohne Sicherheit, sodass jeder mit Zugriff auf die URL alles tun kann. Für einen Test akzeptabel, niemals in Produktion.
- **Kein persistentes Volume**: Jenkins-Daten (`/var/jenkins_home`) gehen verloren, wenn der Pod neu erstellt wird. In Produktion wird ein `PersistentVolumeClaim` eingebunden.
- **Image-Version**: `jenkins/jenkins:lts` folgt der neuesten LTS-Version. In Produktion legt man eine genaue Version fest, um reproduzierbare Deployments zu erhalten.

## Aufräumen

```bash
kubectl delete -f jenkins.yaml
```
