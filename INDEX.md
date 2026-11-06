# Index documentaire Rustkins

Cet index reste à la racine du dépôt. Les six domaines documentaires sont regroupés sous `docs/` ; les décisions, connaissances, recherches, modèles et schémas restent dans leurs dossiers racine.

Cet index permet de localiser la **documentation officielle**, les **décisions** et les **connaissances**. La source de vérité reste chaque fichier Markdown, en tenant compte de son statut et de sa version de produit.

## Versions et périmètre

- [Roadmap des versions](docs/versions/roadmap.md)
- [Inventaire des fonctionnalités](docs/versions/features.md)
- [V0 — Récapitulatif détaillé](docs/versions/v0/overview.md)
- [V1 — Récapitulatif détaillé](docs/versions/v1/overview.md)

## Spécifications YAML des pipelines

Attention : **version de spécification YAML ≠ version du produit**.

| Version du produit | Version YAML | Document |
| --- | --- | --- |
| V0 | v1 | [Spécification YAML v1](docs/specifications/pipeline/yaml-v1.md) |
| V1 | v2 | [Spécification YAML v2](docs/specifications/pipeline/yaml-v2.md) |
| V2 | v3 | [Spécification YAML v3](docs/specifications/pipeline/yaml-v3.md) |

## Architecture et conception technique

- [Runtime et cycle de vie de l'exécution](docs/architecture/components/01-runtime-lifecycle.md)
- [Messagerie RabbitMQ](docs/architecture/components/messaging/02-rabbitmq-architecture.md)
- [Logs en temps réel](docs/architecture/components/logging/03-real-time-logs.md)
- [Artéfacts](docs/architecture/components/storage/04-artifacts.md)
- [Sécurité](docs/architecture/components/security/05-security.md)
- [Scheduling des ressources](docs/architecture/components/scheduler/06-resource-scheduling.md)
- [Git et webhooks](docs/architecture/components/api/07-git-webhooks.md)
- [Versionnement des pipelines](docs/architecture/components/pipeline/08-pipeline-versioning.md)
- [Worker](docs/architecture/components/worker/09-worker-architecture.md)
- [Modèle de données — référence](docs/architecture/data-model/10-data-model-reference.md)
- [Scheduler](docs/architecture/components/scheduler/11-scheduler-architecture.md)
- [MCD V0 (Mermaid)](docs/architecture/data-model/MCD_Rustkins_V0.mmd) · [Image](docs/architecture/data-model/MCD_Rustkins_V0.png)
- [MCD V1 (Mermaid)](docs/architecture/data-model/MCD_Rustkins_V1.mmd) · [Image](docs/architecture/data-model/MCD_Rustkins_V1.png)

## Base de connaissances et règles

- [Décisions d'architecture (ADR)](decisions/README.md)
- [Connaissances et dépannage](knowledge/README.md)
- [Procédures opérationnelles](docs/operations/README.md)
- [Recherche exploratoire et POC](research/README.md)
- [Gestion de projet Agile et Azure Boards](docs/project-management/README.md)
- [Workflow RustkinsActivity et plan d'intégration](docs/project-management/activity-journal-integration.md)
- [Organisation et contribution à la mémoire](docs/conventions/knowledge-management.md)

## Contribuer

- [Template ADR](templates/adr.md) · [Schéma ADR](schemas/adr.schema.json)
- [Template connaissance](templates/knowledge.md) · [Schéma connaissance](schemas/knowledge.schema.json)
- [Template runbook](templates/runbook.md)
- [Template de document](templates/document.md) · [Schéma documentaire](schemas/document.schema.json)

Les répertoires thématiques de connaissances et d'opérations sont créés **à la demande**, sans documents fictifs.
