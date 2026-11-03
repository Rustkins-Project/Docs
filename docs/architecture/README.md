# Architecture Rustkins

Documentation de la **conception du système**, des interactions et des modèles de données. Les raisons des choix structurants sont consignées séparément dans [les ADR](../../decisions/README.md).

## Composants

- [Runtime](components/01-runtime-lifecycle.md)
- [Messaging / RabbitMQ](components/messaging/02-rabbitmq-architecture.md)
- [Logs](components/logging/03-real-time-logs.md)
- [Artifacts](components/storage/04-artifacts.md)
- [Security](components/security/05-security.md)
- [Resource scheduling](components/scheduler/06-resource-scheduling.md)
- [Git webhooks](components/api/07-git-webhooks.md)
- [Pipeline versioning](components/pipeline/08-pipeline-versioning.md)
- [Worker](components/worker/09-worker-architecture.md)
- [Scheduler](components/scheduler/11-scheduler-architecture.md)

## Modèles de données

- [Référence du modèle](data-model/10-data-model-reference.md)
- [MCD V0 (Mermaid)](data-model/MCD_Rustkins_V0.mmd) · [Image](data-model/MCD_Rustkins_V0.png)
- [MCD V1 (Mermaid)](data-model/MCD_Rustkins_V1.mmd) · [Image](data-model/MCD_Rustkins_V1.png)

Les diagrammes accompagnent les spécifications ; ils ne prévalent pas automatiquement sur un contrat plus récent ou une décision approuvée.
