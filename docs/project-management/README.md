# Gestion de projet Rustkins

Référentiel Agile de Rustkins pour l'équipe (Timothy, Alexis et Loick) et les agents Codex / Claude Code. Les règles détaillées sont réparties par sujet ci-dessous ; **chaque règle a un document de référence unique**.

## Trouver une information

| Besoin | Document |
| --- | --- |
| Comprendre les principes, les responsabilités et les validations | [Principes et gouvernance](principles-and-governance.md) |
| Créer et préparer une Epic, Feature, US, Task, Bug ou Issue ; consulter la DoR | [Backlog et Definition of Ready](backlog-and-ready.md) |
| Comprendre `New`, `Active`, `Blocked`, `On Hold`, `Resolved`, `Closed` et les dépendances | [États et dépendances](states-and-dependencies.md) |
| Vérifier les critères de fin d'une Task ou d'une US, les tests et les preuves | [DoD, tests et validation](definition-of-done-and-tests.md) |
| Préparer et conduire les sprints, les reviews, les rétrospectives et les reports | [Processus des sprints](sprint-process.md) |
| Agir sur Azure Boards depuis Codex / Claude Code | [Règles des agents et automatisations](agent-workflow.md) |
| Comprendre le workflow RustkinsActivity et le plan des futurs skills | [Intégration du journal d'activités](activity-journal-integration.md) |
| Retrouver un exemple d'US et les indicateurs | [Exemple et KPI](example-and-kpis.md) |
| Configurer les outils, les intégrations et consulter les références | [Configuration Azure DevOps](azure-devops-configuration.md) |

## Comment utiliser ce référentiel

- **Azure Boards** est la source de vérité pour les work items, leurs états, les priorités et les sprints.
- **RustkinsActivity** conserve le journal des interventions, sans recopier le backlog.
- Les US ne sont `Closed` qu'après validation humaine pendant la **Sprint Review** ; les exigences complètes sont détaillées dans [la DoD](definition-of-done-and-tests.md) et [les règles des sprints](sprint-process.md).
- Les agents doivent rechercher les éléments existants et respecter [leurs règles d'intervention](agent-workflow.md).

Chaque guide dispose de ses propres sections numérotées à partir de 1 et de métadonnées YAML conformes au [schéma documentaire](../../schemas/document.schema.json). Ce `README.md` sert uniquement à la navigation et ne comporte pas de métadonnées.
