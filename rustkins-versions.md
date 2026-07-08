# Rustkins — Roadmap fonctionnelle par versions

# Objectif

Le développement de Rustkins doit être découpé en plusieurs versions cohérentes et finissables afin d'éviter l'accumulation de dette technique et de garantir un socle robuste avant l'ajout de fonctionnalités avancées.

Chaque version doit être considérée comme un produit utilisable et stable.

---

# V0 — Moteur CI local viable

## Objectif

Construire le socle du moteur CI.

À la fin de cette version, Rustkins doit être capable d'exécuter un pipeline complet depuis un dépôt Git jusqu'à l'affichage du résultat final.

## Infrastructure

* API Rustkins
* Scheduler
* Worker
* PostgreSQL
* RabbitMQ
* RustFS
* Docker local (Docker Engine)
* Déploiement via Docker Compose

## Fonctionnalités

### Gestion des projets

* Création d'un projet
* Association d'un dépôt Git
* Déclenchement manuel d'un pipeline

### Moteur de pipeline

* Parser YAML Rustkins
* Validation du schéma YAML
* Gestion des variables simples
* Détection des erreurs de configuration
* Détection des cycles dans le DAG
* Construction du graphe d'exécution

### Exécution

* Clone Git au SHA demandé
* Un conteneur Docker par job
* Exécution séquentielle des steps
* Capture des logs
* Capture du code de retour
* Gestion des dépendances (`needs`)
* Gestion du fail-fast
* Propagation des statuts `skipped`

### Persistance

* Historique des runs
* Historique des jobs
* Historique des steps
* Persistance des logs dans RustFS

### Interface utilisateur

* Liste des pipelines
* Liste des runs
* Détail d'un run
* Affichage des logs après exécution
* Affichage des statuts

## Hors périmètre

* Webhooks Git
* Logs temps réel
* Artifacts avancés
* Cache
* Secrets
* Annulation des jobs
* Scheduling avancé
* Sidecars / Services
* Docker Swarm
* Kubernetes

## Critère de validation

Rustkins doit être capable d'exécuter :

```yaml
jobs:
  build:
    image: node:22
    steps:
      - npm install
      - npm run build

  test:
    image: node:22
    needs: [build]
    steps:
      - npm test
```

---

# V1 — Moteur CI utilisable en équipe

## Objectif

Permettre l'utilisation réelle de Rustkins au sein d'une petite équipe.

## Infrastructure

* Support de plusieurs workers
* Support Docker Swarm
* Déploiement distribué simple

## Fonctionnalités

### Authentification et sécurité

* Authentification JWT
* Gestion des utilisateurs
* Gestion des membres d'un projet
* Tokens d'API par projet

### Déclenchement

* Webhooks Git
* Vérification HMAC

### Exécution

* Annulation d'un run
* Annulation d'un job
* Retry RabbitMQ
* Dead Letter Queues
* Idempotence côté worker

### Logs

* Logs temps réel (SSE)
* Reconnexion automatique aux logs

### Artifacts

* Upload d'artifacts
* Téléchargement d'artifacts
* Politique de rétention

### Pipeline

* Conditions d'exécution :

  * `run_if`
  * `always`
  * `on_failure`
* Warnings non bloquants :

  * jobs optionnels (`optional`)
  * steps non bloquantes (`continue_on_error`)
* Exécution conditionnelle par branche

### Ressources

* Limitation CPU
* Limitation mémoire
* Gestion simple des ressources

## Hors périmètre

* Kubernetes
* Cache avancé
* Secrets complets
* Sidecars
* Matrix builds

## Critère de validation

Un push Git doit automatiquement :

1. Déclencher un pipeline.
2. Afficher les logs en temps réel.
3. Produire des artifacts téléchargeables.
4. Être exécutable sur plusieurs workers.

---

# V2 — Moteur CI/CD distribué

## Objectif

Transformer Rustkins en véritable plateforme CI/CD orientée production.

## Infrastructure

* Docker Swarm complet
* Support Kubernetes
* Déploiement sur plusieurs nœuds

## Fonctionnalités

### Sécurité

* Secrets chiffrés au repos
* Injection sécurisée des secrets
* Masquage des secrets dans les logs

### Autorisations

* Permissions fines
* Gestion des rôles avancée

### Exécution avancée

* Services / sidecars
* Réseaux isolés par job
* Healthchecks
* Exécution E2E complexe

### Cache

* Cache déclaratif
* Réutilisation entre runs
* Invalidation du cache

### Pipeline avancé

* `when: changed`
* Matrix builds statiques
* Fan-in après matrix

### Résilience

* Reprise après crash
* Recovery partiel du scheduler

### Déploiement

* Environnements
* Déploiements Swarm
* Déploiements Kubernetes

## Critère de validation

Rustkins doit être capable d'exécuter un pipeline E2E complexe utilisant :

* base PostgreSQL temporaire ;
* services Docker ;
* secrets ;
* cache ;
* exécution parallèle.

---

# V3 — Plateforme Rustkins avancée

## Objectif

Différencier Rustkins des plateformes CI classiques.

## Fonctionnalités

### Registry interne

* Registry Docker interne
* Images prébuildées Rustkins
* Versionning des images

### Gestion des images

* Interface de création d'images
* Publication dans le registry interne
* Proposition de mise à jour des pipelines

### Réutilisabilité

* Templates de pipelines
* Bibliothèque de jobs réutilisables
* Marketplace interne

### Optimisation

* Statistiques avancées
* Historique des performances
* Suggestions d'optimisation
* Optimisation automatique des pipelines

### Gestion des ressources

* Scheduling intelligent
* Quotas avancés
* Priorités d'exécution

### Multi-tenant

* Isolation renforcée
* Gestion avancée des organisations

## Critère de validation

Rustkins doit proposer une expérience proche des plateformes professionnelles tout en conservant sa simplicité d'utilisation.

---

# Résumé

| Version | Objectif principal                   |
| ------- | ------------------------------------ |
| V0      | Moteur CI local fonctionnel          |
| V1      | Utilisation en équipe                |
| V2      | Plateforme CI/CD distribuée          |
| V3      | Plateforme avancée et différenciante |

# Principe fondamental

Chaque version doit être :

* totalement utilisable ;
* stable ;
* documentée ;
* testée ;
* déployable indépendamment.

Aucune fonctionnalité d'une version supérieure ne doit être commencée tant que la version courante n'est pas terminée.
