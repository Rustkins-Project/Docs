---
id: DOC-0015
title: "Principes et gouvernance de la gestion de projet"
type: project-management
status: approved
summary: "Principes directeurs, sources de vérité, répartition des responsabilités et politique collective de validation."
topics: [governance, agile, roles, backlog]
applies_to: [all]
---

# Principes et gouvernance de la gestion de projet

> **Statut :** règles de gestion de projet Rustkins, utilisables comme référentiel pour les développeurs et les agents IA.  
> **Périmètre :** développement de Rustkins (V0, V1, V2, V3 et versions ultérieures), gestion du backlog, des sprints et des interventions.  
> **Équipe :** Timothy, Alexis et Loick.

## 1. Objectif et principes directeurs

Ce document définit la gestion Agile de Rustkins : hiérarchie des éléments de travail, préparation des sprints, colonnes Azure Boards, responsabilités, critères d'acceptation, **Definition of Ready (DoR)** et **Definition of Done (DoD)**. Il constitue une référence pour l'équipe et un socle métier pour les futurs skills Codex et Claude Code.

1. **Azure Boards est la source officielle** des Epics, Features, US, Tasks, Bugs, Issues, priorités, itérations et états. `RustkinsActivity` conserve le journal des interventions, sans copier le backlog.
2. **Epic = version / lot** ; **Feature = fonctionnalité principale regroupant les US** ; **US = capacité vérifiable** ; **Task = action technique ou organisationnelle suivie**.
3. Une **nouvelle US** n'est jamais créée par un agent sans **validation explicite d'un membre de l'équipe** après recherche des éléments existants.
4. Une **Task** est créée lorsqu'un suivi est justifié, après recherche des Tasks existantes. Les petites interventions peuvent être consignées uniquement dans `RustkinsActivity`.
5. **Les tests font partie intégrante du développement et de la DoD**. Ne pas créer de Tasks séparées pour écrire ou exécuter les tests unitaires, d'intégration ou E2E : les agents les réalisent en continu avec les Tasks de développement.
6. Une US passe à **`Resolved`** seulement lorsque tous ses critères d'acceptation sont respectés, que ses tests unitaires, d'intégration et E2E sont réussis, que toutes ses Tasks sont `Closed` et qu'elle est déployée sur la plateforme de test.
7. Une US passe à **`Closed` pendant la Sprint Review**, après validation humaine de l'équipe. Une US simplement `Resolved` n'est pas encore considérée comme terminée.
8. **`Blocked` et `On Hold` sont à la fois des états Azure DevOps (`System.State`) et des colonnes dédiées du Board**, pour les US et les Tasks. Ils sont créés dans le processus Agile hérité, en catégorie **In Progress**, afin de représenter respectivement une dépendance bloquante et une pause volontaire sans compter le travail comme terminé.
9. **Timothy pilote le Product Backlog et les modifications du Sprint Backlog en cours de sprint.** Les décisions de produit, d'architecture et de validation impliquent l'équipe de trois personnes : Timothy, Alexis et Loick.
10. Aucun agent ne doit inventer une preuve de test, de déploiement ou de validation, ni annoncer une modification Azure/GitHub non confirmée.

L'équipe travaille sur son temps libre : le suivi est prioritairement **asynchrone** via Azure Boards, `RustkinsActivity` et les récapitulatifs Discord ; il n'y a pas de Daily obligatoire.

---

## 2. Gouvernance et responsabilités

**L'équipe projet est constituée de Timothy, Alexis et Loick.** Ces trois membres sont les personnes habilitées à prendre les décisions humaines requises par le projet : création des nouvelles US, validation fonctionnelle, décisions d'architecture, dérogations exceptionnelles et validation des versions. Les agents IA proposent, documentent et vérifient ; ils ne se substituent pas à l'équipe.

| Sujet | Responsabilité |
|---|---|
| Vision, périmètre et choix structurants | Décision de l'équipe : Timothy, Alexis et Loick |
| Rédaction et approbation des nouvelles US | Proposition possible par un agent ; accord explicite d'un membre de l'équipe avant création ; arbitrage de priorité par Timothy |
| Product Backlog : ordre et priorités | **Timothy**, après échanges avec l'équipe |
| Sprint Goal et choix du sprint | Définis en équipe ; coordination du Sprint Backlog par Timothy |
| Modifications du Sprint Backlog pendant un sprint | **Timothy**, avec justification des ajouts, retraits et reports |
| Réalisation des Tasks et tests associés | Développeur responsable, assisté de son agent ; DoD et preuves obligatoires |
| Passage de l'US à `Resolved` | Vérification factuelle et complète de la DoD ; peut être assistée par un agent |
| Validation finale de l'US et passage à `Closed` | **Décision humaine de l'équipe pendant la Sprint Review**, consignée dans son compte rendu |
| Sprint Review / Retrospective | Trois développeurs ; compte rendu distinct pour chaque cérémonie |
| Publication des décisions d'architecture | Validation humaine de l'équipe, suivie d'une trace durable dans `RustkinsDocs` |
| Rapport régulier | Synthèse asynchrone Azure + `RustkinsActivity` + CI/GitHub, sans Daily obligatoire |

**La politique de suivi des Bugs ([§ 1.5](backlog-and-ready.md)) est validée par l’équipe.** Elle rend les régressions visibles sans créer un Bug à chaque échec de test. Les règles de gestion de projet définies dans les différents guides de ce dossier constituent les conventions de travail retenues pour Rustkins.

---

