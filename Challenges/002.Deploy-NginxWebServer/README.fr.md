# Déployer un serveur web Nginx sur Kubernetes (Deployment + Service NodePort)

🌍 [English](README.en.md) · [Deutsch](README.de.md)

## Contexte

Une équipe de développement conçoit un site web statique et souhaite le déployer sur un cluster Kubernetes. L'application doit être **hautement disponible** et **scalable**. L'équipe DevOps choisit donc de créer un Deployment avec plusieurs replicas, exposé à l'extérieur du cluster par un Service de type NodePort.

## Objectifs

- Créer un Deployment `nginx-deployment` :
  - image `nginx:latest` (tag explicite) ;
  - conteneur nommé `nginx-container` ;
  - `3` replicas.
- Créer un Service de type `NodePort` nommé `nginx-service` avec le `nodePort` `30011`.

## Architecture

```
   Client ──▶ <IP d'un nœud>:30011
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
        Deployment : nginx-deployment
```

## Prérequis

- Un cluster Kubernetes fonctionnel.
- L'outil `kubectl` configuré pour accéder au cluster.

## Manifest

Fichier `nginx.yaml` :

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

## Déploiement

```bash
kubectl apply -f nginx.yaml
```

## Vérification

```bash
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx-app
kubectl get svc nginx-service
kubectl get endpoints nginx-service
```

Résultat attendu :

- Deployment en `READY 3/3` ;
- 3 Pods à l'état `Running` ;
- Service `NodePort` avec `80:30011/TCP` ;
- 3 adresses IP listées dans les endpoints.

Test d'accès (optionnel) :

```bash
curl http://<IP-d-un-nœud>:30011
```

La page d'accueil « Welcome to nginx! » doit s'afficher.

## Concepts clés

| Concept | Rôle |
|---|---|
| **Deployment** | Décrit l'état voulu (nombre de replicas, image) et recrée automatiquement les Pods défaillants. |
| **Service** | Point d'entrée stable qui répartit le trafic entre les Pods, dont les IP sont éphémères. |
| **NodePort** | Ouvre un port (plage 30000-32767) sur chaque nœud pour un accès depuis l'extérieur. |
| **Labels / selector** | Mécanisme qui relie le Service aux Pods : le Service route vers les Pods dont les labels correspondent à son `selector`. |

## Exemple et contre-exemple

| ✅ Bon exemple | ❌ Contre-exemple | Conséquence |
|---|---|---|
| `selector` du Service identique au label des Pods (`app: nginx-app`) | Label `app: nginx` côté Pods, `app: nginx-app` côté Service | Endpoints vides : le Service ne route aucun trafic |
| `image: nginx:latest` | `image: nginx` (sans tag) | Tag implicite, moins explicite et non conforme à la consigne |
| `nodePort: 30011` (dans la plage 30000-32767) | `nodePort: 80` | Refusé par l'API : hors de la plage autorisée |
| `selector.matchLabels` identique à `template.labels` | Valeurs différentes | Le Deployment est rejeté par l'API |
| Un Deployment qui gère les Pods | 3 Pods créés à la main | Aucun remplacement automatique en cas de panne, mises à jour manuelles |

> **Astuce de diagnostic** : si le Service ne répond pas, commence par `kubectl get endpoints nginx-service`. Une liste vide indique presque toujours un décalage de labels.

## Nettoyage

```bash
kubectl delete -f nginx.yaml
```
