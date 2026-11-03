# Index documentaire Rustkins

Ce fichier est le point d'entrée de la documentation sous `docs/`. Toutes les catégories, les modèles et les schémas de métadonnées sont rassemblés dans ce dossier.

Cet index permet de localiser la **documentation officielle**, les **décisions** et les **connaissances**. La source de vérité reste chaque fichier Markdown, en tenant compte de son statut et de sa version de produit.

## Versions et périmètre

- [Roadmap des versions](versions/roadmap.md)
- [Inventaire des fonctionnalités](versions/features.md)
- [V0 — Récapitulatif détaillé](versions/v0/overview.md)
- [V1 — Récapitulatif détaillé](versions/v1/overview.md)

## Spécifications YAML des pipelines

Attention : **version de spécification YAML ≠ version du produit**.

| Version du produit | Version YAML | Document |
| --- | --- | --- |
| V0 | v1 | [Spécification YAML v1](specifications/pipeline/yaml-v1.md) |
| V1 | v2 | [Spécification YAML v2](specifications/pipeline/yaml-v2.md) |
| V2 | v3 | [Spécification YAML v3](specifications/pipeline/yaml-v3.md) |

## Architecture et conception technique

- [Runtime et cycle de vie de l'exécution](architecture/components/01-runtime-lifecycle.md)
- [Messagerie RabbitMQ](architecture/components/messaging/02-rabbitmq-architecture.md)
- [Logs en temps réel](architecture/components/logging/03-real-time-logs.md)
- [Artéfacts](architecture/components/storage/04-artifacts.md)
- [Sécurité](architecture/components/security/05-security.md)
- [Scheduling des ressources](architecture/components/scheduler/06-resource-scheduling.md)
- [Git et webhooks](architecture/components/api/07-git-webhooks.md)
- [Versionnement des pipelines](architecture/components/pipeline/08-pipeline-versioning.md)
- [Worker](architecture/components/worker/09-worker-architecture.md)
- [Modèle de données — référence](architecture/data-model/10-data-model-reference.md)
- [Scheduler](architecture/components/scheduler/11-scheduler-architecture.md)
- [MCD V0 (Mermaid)](architecture/data-model/MCD_Rustkins_V0.mmd) · [Image](architecture/data-model/MCD_Rustkins_V0.png)
- [MCD V1 (Mermaid)](architecture/data-model/MCD_Rustkins_V1.mmd) · [Image](architecture/data-model/MCD_Rustkins_V1.png)

## Base de connaissances et règles

- [Décisions d'architecture (ADR)](decisions/README.md)
- [Connaissances et dépannage](knowledge/README.md)
- [Procédures opérationnelles](operations/README.md)
- [Recherche exploratoire et POC](research/README.md)
- [Gestion de projet Agile et Azure Boards](project-management/README.md)
- [Organisation et contribution à la mémoire](conventions/knowledge-management.md)

## Contribuer

- [Template ADR](templates/adr.md) · [Schéma ADR](schemas/adr.schema.json)
- [Template connaissance](templates/knowledge.md) · [Schéma connaissance](schemas/knowledge.schema.json)
- [Template runbook](templates/runbook.md)

Les répertoires thématiques de connaissances et d'opérations sont créés **à la demande**, sans documents fictifs.
