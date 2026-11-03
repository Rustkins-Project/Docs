---
id: DOC-0013
title: "Gestion de la documentation et de la mémoire collective"
type: convention
status: approved
summary: "Sources de vérité, classement, niveaux d'autorité, règles de revue et contribution à la mémoire collective de Rustkins."
topics: [documentation, knowledge, governance, agents]
applies_to: [all]
---

# Gestion de la documentation et de la mémoire collective

## 1. Sources de vérité

| Sujet | Référentiel |
| --- | --- |
| Backlog, sprints, work items, états | Azure Boards |
| Code, commits et Pull Requests | Repositories du code sur GitHub |
| Interventions et suivis individuels | RustkinsActivity |
| Spécifications, architecture et conventions | RustkinsDocs |
| ADR acceptées et connaissance durable | RustkinsDocs |
| Questions et réponses validées sur Discord | RustkinsDiscordBot, avant publication éventuelle dans RustkinsDocs |

Les agents ne doivent pas répliquer le backlog dans les fichiers ni transformer un journal d'activité en connaissance durable.

## 2. Règles de classement

Les domaines documentaires sont regroupés dans `docs/`. L'index `INDEX.md`, les décisions (`decisions/`), les connaissances (`knowledge/`), les recherches (`research/`), les modèles (`templates/`) et les schémas (`schemas/`) restent à la racine. Le `README.md` racine est le point d'entrée général.

- `docs/versions/` : roadmap et synthèse des lots du produit.
- `docs/specifications/` : comportement attendu, contrats et règles.
- `docs/architecture/` : organisation technique, composants et modèles de données.
- `decisions/` : décisions et arbitrages (ADR).
- `knowledge/` : observations vérifiables, solutions et pratiques réutilisables.
- `docs/operations/` : procédures exécutables par étapes (runbooks).
- `docs/conventions/` : conventions de contribution et règles d'équipe.
- `docs/project-management/` : gouvernance Agile, DoD, gestion des sprints et Azure Boards.
- `research/` : études exploratoires et POC sans caractère officiel.

**Ne pas dédoubler les documents par version par défaut.** Utiliser `applies_to` dans les documents de connaissance et les ADR ; conserver des spécifications versionnées lorsque le contrat diffère. La **version YAML d'un pipeline** est distincte de la **version du produit**.

## 3. Niveaux d'autorité

1. Les conventions et spécifications publiées décrivent les règles en vigueur dans leur périmètre.
2. Les ADR `accepted` et fusionnées justifient les décisions d'architecture, en respectant `applies_to` ; une ADR `superseded` ne prescrit plus le comportement actuel.
3. Les connaissances `verified` apportent des preuves et des solutions pratiques, sans modifier implicitement un contrat produit.
4. Les connaissances `candidate`, les ADR `proposed` et les recherches sont informatives ; elles ne valent pas décision officielle.

**En cas de contradiction entre documents officiels**, ne pas choisir silencieusement. Signaler le conflit et proposer une correction ou un arbitrage. Les sources peuvent être historiquement divergentes : leur simple déplacement n'en résout pas le contenu.

## 4. Publication et validation

### Connaissance : sans revue obligatoire

Une note dans `knowledge/` peut être ajoutée ou mise à jour **sans approbation humaine préalable**, y compris par un agent autorisé. Une PR est recommandée pour l'historique et les vérifications, mais **aucune revue de pair n'est exigée**. Un agent ne doit pas prétendre avoir testé une solution s'il ne l'a pas réellement vérifiée ; en cas de doute utiliser `candidate`.

### ADR : une revue humaine minimum

Toute ADR créée, modifiée substantiellement ou remplacée passe par une **PR approuvée par au moins un autre membre humain de l'équipe** (Timothy, Alexis ou Loick ; approbateur différent de l'auteur). Les agents IA ne peuvent pas valider seuls une ADR. Le statut `accepted` n'est opposable qu'une fois le fichier sur `main`, avec approbation de cette PR. Une réponse validée sur Discord est une source, pas une publication automatique.

Ces règles doivent être implémentées dans les protections GitHub de la branche ou des chemins concernés pour être imposées techniquement ; **cette PR n'applique pas de protection serveur**. Éviter une protection imposant une revue à toutes les notes `knowledge/`, contraire à la décision d'équipe.

### Autres documents

Les modifications structurantes des spécifications, de l'architecture et des conventions passent par PR, avec revue adaptée à leur impact. Les procédures opérationnelles ne doivent pas exécuter d'actions dangereuses sans vérification des prérequis.

## 5. Formats Markdown

- Les ADR et les connaissances utilisent leurs schémas respectifs dans `schemas/`.
- Tous les fichiers Markdown sous `docs/` (y compris les README de section) utilisent le frontmatter YAML de `schemas/document.schema.json` et le modèle `templates/document.md`. Les README et l'index à la racine sont des points d'entrée hors de ce périmètre.
- Les métadonnées documentaires incluent `id`, `title`, `type`, `status`, `summary`, `topics` et `applies_to`. Les champs de références et `specification_version` sont facultatifs.
- `applies_to` décrit la version **du produit Rustkins** ; `specification_version` peut décrire la version d'un contrat YAML distinct.
- `draft` est le statut par défaut d'un document historique sans validation explicite ; `approved` exige une validation effective selon les règles d'équipe. Le fait d'être déjà dans `main` ne prouve pas à lui seul cette validation.
- Les identifiants `DOC-xxxx` sont uniques et stables. Le schéma vérifie la structure du frontmatter ; l'unicité, la validité des liens et les autorisations de revue nécessitent des contrôles complémentaires.
- Les schémas JSON s'appliquent **aux objets résultant du parsing YAML**, pas au fichier Markdown brut.
- Le corps doit présenter des faits précis, des limites et des références vérifiables.
- Identifiants stables : `ADR-0001`, `KNOW-0001` (exemples) ; s'assurer qu'ils n'existent pas déjà.
- Renseigner `applies_to` avec les versions du produit concernées ; utiliser `all` si réellement transversal.
- Ne pas inventer de preuve, d'approbation, de résultat CI, ni de statut GitHub/Azure.

## 6. Travail des agents Codex et Claude Code

1. Synchroniser le clone de **lecture** du dépôt avec `git pull --ff-only` en début de session et régulièrement si nécessaire ; ne pas perturber un worktree d'écriture avec des changements locaux.
2. Avant chaque intervention importante, rechercher par **sujet + version**, en priorité dans les spécifications, les ADR acceptées et l'architecture.
3. Lire **seulement les fichiers pertinents**, plutôt que charger toute la base de connaissances.
4. Lorsqu'une découverte mérite une conservation durable, rechercher d'abord la note existante. Proposer son enrichissement ou une création sous `knowledge/`.
5. Pour un changement de conception, proposer une ADR et demander la validation appropriée ; ne pas publier une décision acceptée sans revue humaine.
6. Pour une décision Discord, le skill récupérera la réponse via l'API du bot, préparera l'ADR et sa PR ; il ne considérera pas une réponse en base comme une ADR déjà officielle.
7. Préserver les références vers Azure Boards et GitHub ; éviter les journaux détaillés ou transcriptions.
8. Ne jamais committer des secrets, credentials, données personnelles inutiles ou contenu sensible.

## 7. Maintenance documentaire

Lors d'un déplacement ou d'une réorganisation, vérifier les liens Markdown relatifs, les images et les références croisées. Mettre à jour les liens utilisés par les scripts et autres repositories. Les anciennes URLs vers les fichiers ne sont pas nécessairement redirigées par GitHub.

Ne pas créer de décisions rétrospectives sur la seule base d'un document technique, ni déclarer un ancien brouillon comme validé sans approbation.
