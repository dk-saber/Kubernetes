# Afficher des variables d'environnement dans un Pod Kubernetes

🌍 [English](README.en.md) · [Deutsch](README.de.md)

## Contexte

Une équipe DevOps prépare les prérequis d'une application qui enverra des messages d'accueil personnalisés. Avant de déployer la vraie application, elle teste sur un Pod simple que les variables d'environnement sont correctement injectées dans le conteneur et que leurs valeurs sont celles attendues.

Ce README utilise un **exemple générique** : les noms et les valeurs peuvent être remplacés par les tiens.

## Objectifs

- Créer un Pod nommé `env-demo`.
- Nommer le conteneur `demo-container` et utiliser l'image `bash`.
- Définir trois variables d'environnement :
  - `GREETING` = `Hello from`
  - `APP_NAME` = `Acme`
  - `DEPARTMENT` = `Engineering`
- Utiliser la commande `["/bin/sh", "-c", 'echo "$(GREETING) $(APP_NAME) $(DEPARTMENT)"']`.
- Définir `restartPolicy: Never` pour éviter une boucle de redémarrage (`CrashLoopBackOff`).

## Fonctionnement

```
 env:
 GREETING="Hello from"    ──┐
 APP_NAME="Acme"          ──┼──▶ Kubernetes remplace $(VAR)
 DEPARTMENT="Engineering" ──┘    avant de lancer le conteneur
                                      │
                                      ▼
                     echo "Hello from Acme Engineering"
                                      │
                                      ▼
              kubectl logs ──▶ Hello from Acme Engineering
```

## Prérequis

- Un cluster Kubernetes fonctionnel.
- L'outil `kubectl` configuré pour accéder au cluster.

## Manifest

Fichier `env-demo.yaml` :

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

## Déploiement

```bash
kubectl apply -f env-demo.yaml
```

## Vérification

```bash
kubectl get pod env-demo
kubectl logs env-demo
```

Résultat attendu :

```
Hello from Acme Engineering
```

Le statut du Pod est `Completed` : le conteneur a affiché son message puis s'est terminé normalement, ce qui est le comportement voulu avec `restartPolicy: Never`.

> **Remarque sur `-f`** : `kubectl logs -f` suit les logs en direct. Ici, le conteneur se termine aussitôt, donc `-f` n'apporte rien de plus qu'un simple `kubectl logs`.

## Concepts clés

| Concept | Rôle |
|---|---|
| **Variable d'environnement** | Paire nom/valeur injectée au démarrage pour configurer l'application sans modifier l'image. |
| **`$(VAR)`** | Syntaxe de **Kubernetes** (et non du shell) : remplacée par la valeur de la variable avant le lancement du conteneur, dans `command` et `args`. |
| **`restartPolicy: Never`** | Le Pod n'est pas relancé une fois le conteneur terminé. Par défaut (`Always`), il serait relancé en boucle. |

## Exemple et contre-exemple

| ✅ Bon exemple | ❌ Contre-exemple | Conséquence |
|---|---|---|
| `$(APP_NAME)` avec `APP_NAME` défini dans `env` | `$(APP_NAME)` sans la variable dans `env` | Le texte reste intact et le shell tente d'exécuter une commande `APP_NAME` : `not found` |
| `restartPolicy: Never` au niveau de `spec` du Pod | `restartPolicy` placé dans le conteneur | Champ inconnu, manifest rejeté |
| Valeurs entre guillemets (`"Hello from"`) | Valeurs sans guillemets comme `yes` ou `123` | Risque d'interprétation en booléen ou nombre par YAML |
| Statut `Completed` attendu | S'attendre à un Pod `Running` | Un Pod `Completed` est normal pour une tâche ponctuelle |
| Manifest YAML pour une commande avec guillemets imbriqués | `kubectl run` avec guillemets imbriqués | Le shell local interprète les caractères avant l'envoi |

## Nettoyage

```bash
kubectl delete -f env-demo.yaml
```
