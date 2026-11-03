# RustkinsDocs

Toute la documentation de Rustkins est centralisée sous [`docs/`](docs/INDEX.md). Le `README.md` à la racine sert uniquement de point d'entrée.

Référentiel documentaire et mémoire collective de **Rustkins** : spécifications, architecture, décisions, connaissances réutilisables, procédures et conventions. Les documents Markdown restent la source de vérité : l'index facilite leur découverte mais ne remplace pas leur contenu.

**[Accéder à l'index de la documentation](docs/INDEX.md)** · **[Règles de contribution et gouvernance](docs/conventions/knowledge-management.md)**

## Où ranger une information ?

| Nature de l'information | Dossier | Exemple |
| --- | --- | --- |
| Objectifs et périmètre d'une version | [versions/](docs/versions/README.md) | V0, V1, roadmap |
| Comportement attendu du produit | [specifications/](docs/specifications/README.md) | Syntaxe YAML, règles API |
| Organisation technique des composants | [architecture/](docs/architecture/README.md) | Scheduler, RabbitMQ, modèle de données |
| Arbitrage technique ou architectural | [decisions/](docs/decisions/README.md) | ADR acceptée ou proposée |
| Observation durable et réutilisable | [knowledge/](docs/knowledge/README.md) | Débogage, pièges, patterns |
| Procédure reproductible | [operations/](docs/operations/README.md) | Déploiement, maintenance, incident |
| Gestion Agile, sprints, DoD | [project-management/](docs/project-management/README.md) | Processus Azure Boards et suivi de projet |
| Règle d'équipe ou de contribution | [conventions/](docs/conventions/knowledge-management.md) | Git, qualité, mémoire collective |
| Étude exploratoire sans décision | [research/](docs/research/README.md) | Comparaison, POC |

**Autres sources de vérité :**
- Azure Boards : backlog, états, itérations et tâches.
- RustkinsActivity : journal des interventions (pas un duplicata de cette documentation).
- GitHub (repos du code) : implémentation, commits et PR.
- RustkinsDiscordBot : discussion et validation d'une réponse, avant éventuelle formalisation en ADR.

## Règles essentielles

1. Rechercher un document existant avant d'en créer un nouveau ; mettre à jour la source canonique.
2. Ne pas confondre **version du produit** (V0, V1, V2…) et **version de la syntaxe YAML** (v1, v2, v3…).
3. Préciser la version concernée lorsque le comportement diffère selon les versions.
4. Les connaissances sont publiables **sans revue obligatoire** ; une ADR exige **au moins une approbation d'un autre membre** avant fusion.
5. Les propositions et recherches ne sont pas des décisions officielles.
6. Ne jamais publier de secret, jeton ou transcription privée dans le dépôt.

Les documents historiques sont classés par nature. Utiliser [l'index](docs/INDEX.md) pour retrouver les spécifications et références techniques.
