# Mise à jour progressive et retour arrière d'un Deployment Kubernetes

🌍 [English](README.en.md) · [Deutsch](README.de.md)

## Contexte

Une mise en production est prévue la semaine suivante. Avant de la réaliser, l'équipe DevOps souhaite répéter la **mise à jour** et le **retour arrière (rollback)** d'un Deployment sur un environnement de développement, afin d'identifier les risques à l'avance.

Ce README utilise un **exemple générique** : les noms, le namespace et le port peuvent être remplacés par les tiens.

## Objectifs

- Créer un namespace `demo`.
- Créer un Deployment `web-deploy` dans ce namespace :
  - un conteneur nommé `web`, image `httpd:2.4.28`, `3` replicas ;
  - stratégie `RollingUpdate` avec `maxSurge: 1` et `maxUnavailable: 2`.
- Créer un Service `NodePort` nommé `web-service` exposant le Deployment sur le `nodePort` `30080`.
- Mettre à jour le Deployment vers `httpd:2.4.43` par une mise à jour progressive.
- Une fois tous les Pods à jour, annuler la mise à jour et revenir à la version d'origine.

## Fonctionnement

```
  ① 2.4.28             ② 2.4.43             ③ Rollback
┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│ RS A 2.4.28 ■■■│──▶│ RS A 2.4.28  — │──▶│ RS A 2.4.28 ■■■│
│ RS B  —        │   │ RS B 2.4.43 ■■■│   │ RS B 2.4.43  — │
└────────────────┘   └────────────────┘   └────────────────┘
RS = ReplicaSet · ■ = Pod
```

À chaque modification du template de Pod, le Deployment crée un nouveau ReplicaSet et conserve l'ancien (sans Pods). Le rollback réactive simplement l'ancien ReplicaSet.

## Les paramètres de la stratégie (avec 3 replicas)

| Paramètre | Valeur | Signification | Effet |
|---|---|---|---|
| `maxSurge` | 1 | Nombre de Pods **en plus** autorisés pendant la mise à jour | Jusqu'à 4 Pods simultanés |
| `maxUnavailable` | 2 | Nombre de Pods **pouvant manquer** | Au moins 1 Pod reste disponible |

## Prérequis

- Un cluster Kubernetes fonctionnel.
- L'outil `kubectl` configuré pour accéder au cluster.

## Manifest

Fichier `web.yaml` :

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

## Déroulement

**1. Déploiement initial**

```bash
kubectl apply -f web.yaml
kubectl rollout status deployment/web-deploy -n demo
```

**2. Mise à jour vers 2.4.43** (format `nom-du-conteneur=image`)

```bash
kubectl set image deployment/web-deploy web=httpd:2.4.43 -n demo
kubectl rollout status deployment/web-deploy -n demo
```

**3. Retour arrière** (une fois tous les Pods à jour)

```bash
kubectl rollout undo deployment/web-deploy -n demo
kubectl rollout status deployment/web-deploy -n demo
```

## Vérification

```bash
# Image après le rollback (attendu : httpd:2.4.28)
kubectl get deployment web-deploy -n demo -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

kubectl get pods -n demo
kubectl get svc -n demo
kubectl get rs -n demo
kubectl rollout history deployment/web-deploy -n demo

# Version réellement servie (attendu : Apache/2.4.28)
curl -sI http://<IP-d-un-nœud>:30080 | grep -i server
```

Résultat attendu : image `httpd:2.4.28`, 3 Pods `Running`, Service `80:30080/TCP`.

> **Historique** : après le rollback, tu verras les révisions **2 et 3**, et plus la 1. Le contenu de l'ancienne révision 1 est reclassé comme nouvelle révision 3. C'est le comportement normal de Kubernetes.

## Concepts clés

| Concept | Rôle |
|---|---|
| **RollingUpdate** | Remplace les Pods progressivement, sans interruption totale du service. |
| **ReplicaSet** | Gère les Pods d'une version donnée ; un nouveau est créé à chaque changement du template. |
| **`rollout undo`** | Réactive le ReplicaSet de la révision précédente. |
| **NodePort** | Ouvre un port (plage 30000-32767) sur chaque nœud pour un accès externe. |

## Exemple et contre-exemple

| ✅ Bon exemple | ❌ Contre-exemple | Conséquence |
|---|---|---|
| Créer le Deployment directement avec l'image voulue | Créer avec une autre image puis corriger | Révisions parasites dans l'historique |
| `-n demo` sur chaque commande | Oublier `-n demo` | Ressource introuvable, ou créée dans `default` |
| `set image ... web=httpd:2.4.43` (nom du conteneur) | `web-deploy=httpd:2.4.43` (nom du Deployment) | Erreur : conteneur introuvable |
| Attendre `rollout status` avant le `undo` | Lancer `undo` en pleine mise à jour | Mélange d'anciens et de nouveaux Pods pendant le rollback |
| Stratégie `RollingUpdate` | Stratégie `Recreate` | Tous les Pods sont arrêtés avant les nouveaux : coupure de service |
| `nodePort: 30080` (plage 30000-32767) | Un port hors plage | Refusé par l'API |

> **Attention** : `kubectl set image` ne modifie pas le fichier YAML. Ré-appliquer ensuite `kubectl apply -f web.yaml` ramène l'image à `2.4.28` et crée une révision supplémentaire.

## Nettoyage

```bash
kubectl delete -f web.yaml
```
