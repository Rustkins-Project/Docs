---
id: DOC-0032
title: "Intégration du journal RustkinsActivity au workflow de l'équipe"
type: project-management
status: draft
summary: "Règles transverses entre Azure Boards, GitHub, RustkinsActivity et les agents IA ; scénarios et plan de mise en œuvre du workflow hybride."
topics: [activity, agents, github, azure-boards, automation]
applies_to: [all]
references:
  - "https://github.com/Rustkins-Project/RustkinsActivity"
  - "https://github.com/Rustkins-Project/RustkinsDiscordBot"
---

# Intégration de RustkinsActivity au workflow de l'équipe

> **Document en `draft` jusqu'à validation humaine.** Il synthétise les décisions d'équipe et planifie leur automatisation ; les outils de publication et les skills décrits ici ne sont pas encore développés. Le **contrat canonique du journal** demeure dans [RustkinsActivity](https://github.com/Rustkins-Project/RustkinsActivity).

## 1. Sources de vérité

| Système | Responsabilité |
| --- | --- |
| Azure Boards | Epics, Features, US, Tasks, Bugs, sprints, états officiels, DoD et validation |
| Dépôts GitHub | Code, commits, PR et reviews |
| RustkinsDocs | Spécifications, ADR, conventions et connaissance durable |
| RustkinsActivity | Chronologie, résultats et blocages des interventions individuelles (humains et agents) |
| RustkinsDiscordBot | Questions à l'équipe et récapitulatif quotidien des activités **publiées sur `main`** |

**Une activité n'est pas une US, une Task Azure, une PR ou une session IA**. Une intervention cohérente d'un responsable peut impliquer plusieurs dépôts (`repositories`), plusieurs PR, plusieurs jours ou plusieurs sessions Codex/Claude Code. Une Task peut avoir plusieurs activités ; une PR peut être citée par plusieurs activités. Le champ `work_item_id` vise le work item Azure principal ou vaut `null`, sans création automatique de ticket.

Les règles Azure existantes restent souveraines : une Task ne devient `Closed` qu'après sa DoD, et une US ne devient `Closed` qu'après validation humaine pendant la Sprint Review. Le statut d'une activité ne doit **jamais déclencher automatiquement** de changement d'état Azure. Les PR continuent d'utiliser `AB#123` pour les liaisons Azure reconnues par l'application Azure Boards for GitHub ; les liens d'activité ne remplacent pas cette convention.

## 2. Granularité, reprise et clôture

- **Une activité = une intervention cohérente par responsable humain** (Timothy, Alexis ou Loick). La durée ou le nombre de dépôts ne constitue pas un critère de découpage.
- Énoncer dès le début objectif, périmètre et **condition de fin**. Une évolution cohérente est autorisée tant que l'activité est ouverte, avec modification du Résumé et trace au Journal. Un nouveau travail indépendant exige une autre activité.
- Rechercher en priorité les **brouillons locaux non publiés**, puis les activités ouvertes du responsable : si correspondance claire, reprise automatique par l'agent ; ambiguïté réelle, demander confirmation. Même work item/dépôt ≠ même activité.
- Statuts de journal : `in_progress`, `paused` (suspension volontaire), `blocked` (dépendance vérifiée), `completed` et `abandoned`. Une fin de conversation ne change pas le statut.
- **Immutabilité totale** dès la publication de `completed` ou `abandoned`. Ni correction, ni nouvelle PR, ni réouverture sur l'ancienne fiche : nouvelle activité et référence à l'original si nouveau travail ou erratum.
- Par défaut, clôturer quand le travail prévu est réalisé, vérifié et le livrable publié (par exemple une PR ouverte). **Merge/review ne sont attendus que si le périmètre l'exige explicitement**. Une review après clôture donnant lieu à du travail crée donc une nouvelle activité, même liée à la même PR.
- Une correction très courte peut produire **directement une activité `completed`**, avec les vraies dates. Pour une intervention significative, créer/publier `in_progress` dès le démarrage et mettre à jour aux étapes utiles.
- Un blocage sur Discord se note dans le journal avec contexte et action attendue ; le retour à `in_progress` n'intervient qu'une fois le blocage réellement levé.

## 3. Politique de publication Git hybride

Les fiches Markdown d'activité sous `activities/` sont publiées directement sur `main` après vérification locale. **Schéma, template, conventions, scripts et hooks** passent par branche + **PR et relecture humaine** ; l'équipe ne met pas en place de CI ni de règles de branches complexes pour ce journal à ce stade.

Chaque développeur dispose **d'un clone local RustkinsActivity partagé par Codex et Claude Code**. Les agents préparent des **brouillons d'opérations ciblées hors du clone**. Un outil commun (futur) détient un verrou **interprocessus court** uniquement pendant la transaction de publication.

Séquence envisagée : prévalidation du brouillon → verrou → état Git connu → `fetch` et `pull --ff-only` → vérification de version du fichier → application ciblée → indexation du seul fichier → validation complète et `pre-commit` → commit → push immédiat → vérification sur `origin/main` → libération du verrou.

- **Pas de pull automatique chaque minute**, pas de `git add .` aveugle, de `--no-verify`, de `reset --hard` automatique, ni de `push --force`.
- Une modification locale inconnue ou un état Git ambigu bloque la transaction sans suppression.
- Le verrou ne couvre pas les autres postes : en cas de push non-fast-forward, récupérer `origin/main`, puis **reconstituer** le commit depuis le brouillon si les opérations restent compatibles. Sinon, signaler un conflit.
- Les brouillons ont un ID stable, un contexte de base et un état local récupérable. Un commit local n'est **pas** une publication : seul `origin/main` confirme l'état `published`.
- Les opérations ciblées sont `create`, `append_journal`, `add_reference`, `add_repository`, `set_status`, `replace_section` et `set_metadata`. Elles sont idempotentes lorsque cela est sûr et appliquées atomiquement par activité. Aucune **deuxième recherche sémantique de doublons sous verrou** ; on conserve les contrôles techniques de concurrence.
- Le `pre-commit` local est obligatoire **par convention d'agent**, non par GitHub. Sans CI ni protection serveur, une écriture manuelle qui contourne le hook n'est pas empêchée ; ce compromis est accepté.

Voir les règles détaillées dans [le cycle de vie](https://github.com/Rustkins-Project/RustkinsActivity/blob/main/docs/activity-workflow.md) et [le contrat de publication](https://github.com/Rustkins-Project/RustkinsActivity/blob/main/docs/publication-contract.md). Ne pas dupliquer la spécification d'outillage dans RustkinsDocs.

## 4. Scénarios bout en bout

| Scénario | Trajet attendu |
| --- | --- |
| Petite correction sans ticket | Recherche → correction et vérifications → `create completed` → hook + commit + push → digest du bot |
| US sur plusieurs jours | Work item Azure existant si pertinent → `in_progress` → plusieurs publications du journal/tests/PR → clôture à la condition de fin |
| Session IA interrompue | Brouillon conservé → reprise par le même agent ou l'autre → pas de nouvel `activity_id` |
| Plusieurs développeurs, même sujet | Une activité par responsable, éventuel même `work_item_id` et mêmes références PR |
| Question Discord bloquante | `blocked` + URL/contexte → vérifier la réponse → `in_progress` puis progression |
| Review après clôture | Nouvelle activité liée à l'ancienne, même PR possible, ancienne immuable |
| Intervention sur plusieurs dépôts | Une activité, tableau `repositories` non vide, références aux PR concernées |

## 5. Plan d'implémentation transversal

| Phase | Dépôt | Livrable et validation attendue |
| --- | --- | --- |
| **0 — Contrat** | **RustkinsActivity** | `repository` → `repositories` (min. 1 dépôt unique), template et documentation. Tester exemples mono/multidépôts. |
| **1 — Validation** | **RustkinsActivity** | Validateur YAML/frontmatter JSON Schema, cohérence chemin/identité/dates/statuts et immutabilité ; hook `pre-commit` installé par setup. Tests sur les versions **indexées**. |
| **2 — Brouillons** | **RustkinsActivity** | Opérations ciblées idempotentes, état local hors clone, comparaison par `blob_sha`, hash de section et transition attendue. Tests unitaires de conflits. |
| **3 — Git/recovery** | **RustkinsActivity** | Verrou OS, synchronisation FF, validation, commit/push, récupération après crash, rejet de push distant ; tests de deux agents / deux clones. |
| **4 — Skills** | **Futur dépôt de skills** | Adaptateurs Codex et Claude Code vers un outil commun, configuration des trois profils ; recherche/reprise et usage transparent. **Aucun dépôt créé maintenant.** |
| **5 — Bot (option)** | **RustkinsDiscordBot** | Ignorer et signaler individuellement une fiche invalide plutôt que perdre tout le récapitulatif ; liens enrichis vers activités. |

Le futur dépôt de skills contiendra uniquement la **distribution et les instructions pour les agents**, pas une seconde implémentation des règles de Git ou de validation. Interface conceptuelle à considérer : `find`, `prepare`, `validate`, `publish`, `recover`, `status`. Éviter service permanent, base de données ou ordonnancement par minute.

## 6. Recette et exclusions

Les tests de bout en bout doivent couvrir : validation d'un tableau multi-dépôts, frontmatter invalide, activité directement terminale, évolution d'objectif tracée, reprise automatique après interruption, publication concurrente sur deux fichiers, collision de statut sur un même fichier, événement/PR sans doublons, crash avant/après commit, timeout de push, refus de modification d'une fiche terminale, blocage Discord et séparation Azure/Activity.

Les critères d'acceptation du chantier : un journal publiable depuis les deux agents sans duplications intentionnelles, aucune écriture concurrente silencieusement perdue, aucun faux succès GitHub, aucune activité terminale modifiée, aucune clôture Azure implicite, et une lecture par le bot de `main` préservée.

**Hors périmètre de ces PR** : développement des scripts, de la CLI, des skills, du hook, d'une CI et de règles de branches GitHub complexes. Les nouveaux liens vers RustkinsActivity seront disponibles sur `main` après fusion de sa PR complémentaire.

## 7. Sources

- [RustkinsActivity — convention et cycle de vie](https://github.com/Rustkins-Project/RustkinsActivity/blob/main/docs/activity-workflow.md)
- [RustkinsActivity — contrat brouillons/Git](https://github.com/Rustkins-Project/RustkinsActivity/blob/main/docs/publication-contract.md)
- [RustkinsActivity — plan d'implémentation et tests](https://github.com/Rustkins-Project/RustkinsActivity/blob/main/docs/implementation-plan.md)
- [Règles des agents](agent-workflow.md) · [DoD Azure](definition-of-done-and-tests.md) · [Gouvernance](principles-and-governance.md)
