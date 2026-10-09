# Déployer Jenkins sur Kubernetes

🌍 [English](README.en.md) · [Deutsch](README.de.md)

## Contexte

Une équipe DevOps souhaite mettre en place un serveur d'intégration continue (CI) **Jenkins** pour créer et gérer les pipelines de ses projets. Plutôt que de l'installer sur une machine dédiée, elle l'héberge sur son cluster Kubernetes, ce qui donne une installation reproductible (un fichier YAML) et un redémarrage automatique en cas de panne.

Ce README utilise un **exemple générique** : les noms, le namespace et le port peuvent être remplacés par les tiens.

## Objectifs

- Créer un namespace `ci-tools`.
- Créer un Deployment `jenkins` dans ce namespace :
  - label `app: jenkins`, `1` replica ;
  - conteneur nommé `jenkins`, image `jenkins/jenkins:lts`, port `8080` ;
  - variable d'environnement `JAVA_OPTS` = `-Djenkins.install.runSetupWizard=false` pour ignorer l'assistant d'installation.
- Créer un Service `NodePort` nommé `jenkins-svc` exposant Jenkins sur le `nodePort` `30090`.
- Attendre que le Pod soit `Running`, puis vérifier l'accès à l'interface dans le navigateur.

## Architecture

```
 Navigateur ──▶ <IP d'un nœud>:30090
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
   Deployment : jenkins (1 replica) · Namespace : ci-tools
```

## Prérequis

- Un cluster Kubernetes fonctionnel.
- L'outil `kubectl` configuré pour accéder au cluster.

## Manifest

Fichier `jenkins.yaml` :

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

## Déploiement

```bash
kubectl apply -f jenkins.yaml
kubectl get pods -n ci-tools -w
```

Attends que le Pod soit `1/1 Running` (`Ctrl+C` pour quitter le mode `-w`).

## Vérification

```bash
kubectl get all -n ci-tools
kubectl logs -n ci-tools deployment/jenkins | tail -5
kubectl get nodes -o wide
curl -I http://<IP-d-un-nœud>:30090
```

Résultat attendu :

- Pod `1/1 Running`, Service `8080:30090/TCP` ;
- dans les logs, un message du type `Jenkins is fully up and running` ;
- dans le navigateur, le tableau de bord Jenkins sur `http://<IP-d-un-nœud>:30090`.

> **Patience** : un Pod `Running` ne veut pas dire que Jenkins est prêt. Au démarrage, il charge ses plugins pendant 1 à 2 minutes et affiche « Please wait while Jenkins is getting ready to work ». Attends que la page se recharge d'elle-même.

## Concepts clés

| Concept | Rôle |
|---|---|
| **Namespace** | Espace logique qui isole les ressources d'un projet du reste du cluster. |
| **Deployment** | Maintient le nombre voulu de Pods Jenkins et recrée le Pod en cas de panne. |
| **Service NodePort** | Point d'entrée stable, accessible depuis l'extérieur via un port (plage 30000-32767) ouvert sur chaque nœud. |
| **`JAVA_OPTS`** | Transmet des options à la JVM ; `runSetupWizard=false` désactive l'assistant d'installation initial. |

## Exemple et contre-exemple

| ✅ Bon exemple | ❌ Contre-exemple | Conséquence |
|---|---|---|
| `-n ci-tools` sur chaque commande | Oublier `-n ci-tools` | « No resources found » : tu regardes le namespace `default` |
| `selector` du Service = label des Pods (`app: jenkins`) | Label différent côté Pods | Endpoints vides, aucune réponse dans le navigateur |
| `targetPort: 8080` (port de Jenkins) | `targetPort: 80` | Connexion refusée : Jenkins n'écoute pas sur 80 |
| Option `JAVA_OPTS` exacte, entre guillemets | Faute de frappe dans l'option | L'assistant « Unlock Jenkins » s'affiche quand même |
| Patienter 1 à 2 minutes après `Running` | Tester l'URL immédiatement | Page d'attente ou erreur 503 temporaire |
| `nodePort` libre sur le cluster | Un NodePort déjà pris, même dans un autre namespace | Refusé : les NodePorts sont uniques à l'échelle du **cluster** |

## Limites à connaître

- **Aucune authentification** : avec `runSetupWizard=false`, Jenkins démarre sans sécurité, donc toute personne ayant accès à l'URL peut tout faire. Acceptable pour un test, jamais en production.
- **Aucun volume persistant** : les données de Jenkins (`/var/jenkins_home`) sont perdues si le Pod est recréé. En production, on monte un `PersistentVolumeClaim`.
- **Version de l'image** : `jenkins/jenkins:lts` suit la version LTS la plus récente. En production, on fixe une version précise pour des déploiements reproductibles.

## Nettoyage

```bash
kubectl delete -f jenkins.yaml
```
