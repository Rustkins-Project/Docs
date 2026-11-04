# Gestion de projet Rustkins

Référentiel Agile de Rustkins pour l'équipe (Timothy, Alexis et Loick) et les agents Codex / Claude Code. Les règles détaillées sont réparties par sujet ci-dessous ; **chaque règle a un document de référence unique**.

## Trouver une information

| Besoin | Document | Sections d'origine |
| --- | --- | --- |
| Comprendre les principes, les responsabilités et les validations | [Principes et gouvernance](principles-and-governance.md) | § 1, § 12 |
| Créer et préparer une Epic, Feature, US, Task, Bug ou Issue ; consulter la DoR | [Backlog et Definition of Ready](backlog-and-ready.md) | § 2, § 4 |
| Comprendre `New`, `Active`, `Blocked`, `On Hold`, `Resolved`, `Closed` et les dépendances | [États et dépendances](states-and-dependencies.md) | § 3 |
| Vérifier les critères de fin d'une Task ou d'une US, les tests et les preuves | [DoD, tests et validation](definition-of-done-and-tests.md) | § 5, § 6 |
| Préparer et conduire les sprints, les reviews, les rétrospectives et les reports | [Processus des sprints](sprint-process.md) | § 7 |
| Agir sur Azure Boards depuis Codex / Claude Code | [Règles des agents et automatisations](agent-workflow.md) | § 8 |
| Retrouver un exemple d'US et les indicateurs | [Exemple et KPI](example-and-kpis.md) | § 9, § 10 |
| Configurer les outils, les intégrations et consulter les références | [Configuration Azure DevOps](azure-devops-configuration.md) | § 11, § 13 |

## Comment utiliser ce référentiel

- **Azure Boards** est la source de vérité pour les work items, leurs états, les priorités et les sprints.
- **RustkinsActivity** conserve le journal des interventions, sans recopier le backlog.
- Les US ne sont `Closed` qu'après validation humaine pendant la **Sprint Review** ; les exigences complètes sont détaillées dans [la DoD](definition-of-done-and-tests.md) et [les règles des sprints](sprint-process.md).
- Les agents doivent rechercher les éléments existants et respecter [leurs règles d'intervention](agent-workflow.md).

Le découpage reprend l'intégralité des **13 sections** du référentiel initial : leurs numéros sont conservés pour faciliter les références historiques. Les fichiers de référence disposent chacun d'un frontmatter YAML conforme au [schéma documentaire](../../schemas/document.schema.json), tandis que ce `README.md` de navigation n'a pas de métadonnées.
