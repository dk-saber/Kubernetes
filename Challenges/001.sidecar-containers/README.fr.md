# Pattern Sidecar sur Kubernetes : expédier les logs de nginx

🌍 [English](README.en.md) · [Deutsch](README.de.md)

## Contexte

Un serveur web nginx produit des logs d'accès et d'erreur. Ces logs ne sont pas assez critiques pour justifier un volume persistant, mais les équipes de développement doivent pouvoir consulter les dernières 24 heures afin de diagnostiquer bugs et incidents. Il faut donc les envoyer vers un service d'agrégation de logs.

En suivant le principe de **séparation des responsabilités**, nginx se limite à servir des pages web, et un second conteneur (le **sidecar**) se charge uniquement d'expédier les logs. Les deux conteneurs vivent dans le même Pod et échangent les fichiers via un volume `emptyDir` partagé.

## Objectifs

- Créer un Pod nommé `webserver`.
- Déclarer un volume `emptyDir` nommé `shared-logs`.
- Ajouter un conteneur applicatif `nginx-container` (image `nginx:latest`).
- Ajouter un conteneur sidecar `sidecar-container` (image `ubuntu:latest`) qui exécute :
  `sh -c "while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done"`
- Monter `shared-logs` dans les deux conteneurs sur `/var/log/nginx`.
- Vérifier que tous les conteneurs sont à l'état `Running`.

## Architecture

```
┌──────────────────── Pod : webserver ────────────────────┐
│                                                          │
│  ┌──────────────────┐            ┌───────────────────┐   │
│  │  nginx-container │  écrit     │  sidecar-container│   │
│  │  (nginx:latest)  │──────┐     │  (ubuntu:latest)  │   │
│  └──────────────────┘      │ ┌──▶│  lit et expédie   │   │
│                            ▼ │   └───────────────────┘   │
│                  ┌──────────────────┐                    │
│                  │ emptyDir         │                    │
│                  │ shared-logs      │                    │
│                  │ /var/log/nginx   │                    │
│                  └──────────────────┘                    │
└──────────────────────────────────────────────────────────┘
```

## Prérequis

- Un cluster Kubernetes en version **1.28 ou supérieure** (sidecars natifs, activés par défaut à partir de la 1.29).
- L'outil `kubectl` configuré pour accéder au cluster.

## Manifest

Fichier `webserver.yaml` :

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
      restartPolicy: Always        # transforme l'init container en sidecar natif
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

## Déploiement

```bash
kubectl apply -f webserver.yaml
kubectl get pod webserver
```

Résultat attendu :

```
NAME        READY   STATUS    RESTARTS   AGE
webserver   2/2     Running   0          10s
```

## Vérification

```bash
# Logs du sidecar (une itération toutes les 30 secondes)
kubectl logs webserver -c sidecar-container

# Contenu du volume partagé
kubectl exec webserver -c sidecar-container -- ls -l /var/log/nginx

# Générer une ligne d'access log (optionnel)
kubectl exec webserver -c nginx-container -- curl -s localhost -o /dev/null
```

> **Remarque** : lors de la toute première itération, le sidecar peut afficher `No such file or directory`, car il démarre avant nginx. Ce n'est pas bloquant : les fichiers apparaissent dès que nginx est lancé et sont lus à l'itération suivante.

## Concepts clés

| Concept | Rôle |
|---|---|
| **Sidecar** | Conteneur auxiliaire qui accompagne le conteneur principal dans le même Pod. |
| **emptyDir** | Volume temporaire créé avec le Pod et supprimé avec lui. Idéal pour des données non critiques. |
| **Sidecar natif** | Init container avec `restartPolicy: Always` : il démarre en premier mais reste actif pendant toute la vie du Pod. |

## Exemple et contre-exemple

| ✅ Bon exemple | ❌ Contre-exemple |
|---|---|
| Init container avec `restartPolicy: Always` : le Pod passe en `2/2 Running`. | Init container classique avec une boucle infinie, sans `restartPolicy` : il ne se termine jamais, le Pod reste bloqué en `Init:0/1` et nginx ne démarre pas. |
| Un conteneur par responsabilité (servir / expédier). | Un seul conteneur qui fait nginx et l'envoi des logs : un plantage du script peut impacter le serveur web. |
| Volume monté au même `mountPath` dans les deux conteneurs. | Chemins de montage différents : le sidecar ne voit jamais les fichiers écrits par nginx. |

## Nettoyage

```bash
kubectl delete pod webserver
```
