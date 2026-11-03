# Migration documentaire vers la mémoire collective

Cette réorganisation **ne change pas le contenu** des documents existants. Les fichiers Mermaid et leurs images sont déplacés avec leurs blobs Git d'origine. Les chemins sont regroupés par nature documentaire.

| Ancien chemin | Nouveau chemin |
| --- | --- |
| `MCD/MCD_Rustkins_V0.mmd` | [`architecture/data-model/MCD_Rustkins_V0.mmd`](architecture/data-model/MCD_Rustkins_V0.mmd) |
| `MCD/MCD_Rustkins_V0.png` | [`architecture/data-model/MCD_Rustkins_V0.png`](architecture/data-model/MCD_Rustkins_V0.png) |
| `MCD/MCD_Rustkins_V1.mmd` | [`architecture/data-model/MCD_Rustkins_V1.mmd`](architecture/data-model/MCD_Rustkins_V1.mmd) |
| `MCD/MCD_Rustkins_V1.png` | [`architecture/data-model/MCD_Rustkins_V1.png`](architecture/data-model/MCD_Rustkins_V1.png) |
| `Pipeline_Specification/Pipeline_Specification_V0.md` | [`specifications/pipeline/yaml-v1.md`](specifications/pipeline/yaml-v1.md) |
| `Pipeline_Specification/Pipeline_Specification_V1.md` | [`specifications/pipeline/yaml-v2.md`](specifications/pipeline/yaml-v2.md) |
| `Pipeline_Specification/Pipeline_Specification_V2.md` | [`specifications/pipeline/yaml-v3.md`](specifications/pipeline/yaml-v3.md) |
| `Recap/Recap_Rustkins_V0.md` | [`versions/v0/overview.md`](versions/v0/overview.md) |
| `Recap/Recap_Rustkins_V1.md` | [`versions/v1/overview.md`](versions/v1/overview.md) |
| `Technical_Specification/01-runtime-lifecycle.md` | [`architecture/components/01-runtime-lifecycle.md`](architecture/components/01-runtime-lifecycle.md) |
| `Technical_Specification/02-rabbitmq-architecture.md` | [`architecture/components/messaging/02-rabbitmq-architecture.md`](architecture/components/messaging/02-rabbitmq-architecture.md) |
| `Technical_Specification/03-real-time-logs.md` | [`architecture/components/logging/03-real-time-logs.md`](architecture/components/logging/03-real-time-logs.md) |
| `Technical_Specification/04-artifacts.md` | [`architecture/components/storage/04-artifacts.md`](architecture/components/storage/04-artifacts.md) |
| `Technical_Specification/05-security.md` | [`architecture/components/security/05-security.md`](architecture/components/security/05-security.md) |
| `Technical_Specification/06-resource-scheduling.md` | [`architecture/components/scheduler/06-resource-scheduling.md`](architecture/components/scheduler/06-resource-scheduling.md) |
| `Technical_Specification/07-git-webhooks.md` | [`architecture/components/api/07-git-webhooks.md`](architecture/components/api/07-git-webhooks.md) |
| `Technical_Specification/08-pipeline-versioning.md` | [`architecture/components/pipeline/08-pipeline-versioning.md`](architecture/components/pipeline/08-pipeline-versioning.md) |
| `Technical_Specification/09-worker-architecture.md` | [`architecture/components/worker/09-worker-architecture.md`](architecture/components/worker/09-worker-architecture.md) |
| `Technical_Specification/10-data-model-reference.md` | [`architecture/data-model/10-data-model-reference.md`](architecture/data-model/10-data-model-reference.md) |
| `Technical_Specification/11-scheduler-architecture.md` | [`architecture/components/scheduler/11-scheduler-architecture.md`](architecture/components/scheduler/11-scheduler-architecture.md) |
| `rustkins-features.md` | [`versions/features.md`](versions/features.md) |
| `rustkins-versions.md` | [`versions/roadmap.md`](versions/roadmap.md) |

## Compatibilité des liens

Les liens Markdown relatifs dans les documents déplacés ont été inspectés : aucun lien local Markdown nécessitant une modification de chemin n'a été trouvé dans ces sources. Les références externes (liens directs GitHub, clones, scripts ou wiki) vers les anciens chemins doivent être mises à jour par leurs détenteurs ; GitHub ne garantit pas une redirection des URLs vers chaque ancien fichier.

Le document \`docs/rustkins-mcp-smoke-test.md\` et la convention de projet existante ne sont pas modifiés dans ce chantier. Aucun contenu de type ADR ou connaissance n'est inféré rétroactivement à partir des spécifications techniques.

## Pour retrouver un document

Consulter [INDEX.md](INDEX.md), la [roadmap](versions/roadmap.md) et les [règles de contribution](conventions/knowledge-management.md).
