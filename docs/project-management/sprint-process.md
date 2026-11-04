---
id: DOC-0028
title: "Organisation des sprints et rituels Agile"
type: project-management
status: approved
summary: "Product Backlog, refinement, Sprint Planning, suivi asynchrone, Sprint Review, rétrospective et report des US."
topics: [sprint, planning, review, retrospective]
applies_to: [all]
---

# Organisation des sprints et rituels Agile

## 1. Gestion du Product Backlog, des Sprints et de l'Agilité

### 1.1. Product Backlog

Le Product Backlog est maintenu et ordonné par **Timothy**, avec les propositions et l'expertise de **Timothy, Alexis et Loick**. Il contient les US des lots Rustkins V0 et suivants, les Bugs suivis et les investigations utiles. Les priorités tiennent compte de la valeur, des dépendances, des risques et de la capacité réelle de l'équipe.

Les Epics représentent les **versions**, pas les sprints. Une US hérite du périmètre de version par son Epic parent, tandis que le champ `Iteration Path` indique le sprint où elle est engagée. Les sprints peuvent traverser plusieurs Features.

### 1.2. Raffinement

L'équipe clarifie les US, vérifie leur INVEST/DoR, identifie les dépendances, les découpe en unités livrables lorsque nécessaire, estime leurs Story Points et évalue leur Business Value. **Timothy ordonne le backlog** et arbitre la sélection des US ; chaque création d'US proposée par un agent nécessite une validation humaine explicite préalable.

**Éviter le surdécoupage :** backend, interface, base de données et infrastructure sont le plus souvent des Tasks d'une même US si elles concourent au même comportement observable. **Les tests ne sont pas des Tasks séparées** : ils accompagnent chaque Task de développement et conditionnent la DoD globale de l'US.

### 1.3. Sprint Planning

1. Définir en équipe le **Sprint Goal**, simple et vérifiable.
2. Évaluer les disponibilités réelles des trois développeurs pour la période : Rustkins est un projet personnel ; les temps libres peuvent varier d'un sprint à l'autre.
3. Sélectionner les US répondant à la DoR, selon priorité, dépendances et capacité ; utiliser les Story Points comme mesure relative, **sans estimation en heures des US ou des Tasks**.
4. Décomposer les US retenues en Tasks techniques ou organisationnelles utiles, avec responsables et résultats attendus. Les tests sont compris dans ces Tasks et dans la DoD.
5. Associer les work items au bon sprint via `Iteration Path` et relire les liens parents/dépendances.
6. Signaler les risques et convenir des actions préventives.

### 1.4. Pendant le sprint

- Le **Sprint Goal** reste stable ; des ajustements de Tasks peuvent être nécessaires.
- **Timothy est responsable des modifications du Sprint Backlog en cours de sprint** : ajout/retrait d'US, changement de priorité, transfert d'un Bug ou ajustement de périmètre. Les agents proposent et justifient ; Timothy décide et en consigne la raison et l'impact.
- Les Tasks supplémentaires sont créées uniquement si leur suivi apporte de la valeur, **jamais pour matérialiser automatiquement l'écriture des tests**.
- Un Bug peut être enregistré à tout moment ; sa prise en charge dans le sprint est arbitrée par Timothy, en concertation avec l'équipe lorsque l'objectif de sprint est menacé.
- Chaque développeur et son agent actualisent Azure Boards et leurs fiches dans `RustkinsActivity` à mesure que le travail progresse ; les résultats CI et les PR fournissent les preuves techniques.
- Les états et colonnes `Blocked` et `On Hold` rendent visibles respectivement les dépendances bloquantes et les mises en pause.

### 1.5. Suivi asynchrone, sans Daily obligatoire

Rustkins est développé par **trois personnes sur leur temps libre**, sans présence ni activité quotidienne garantie. Une cérémonie Daily systématique serait peu utile et risquerait de mobiliser du temps sans apporter de progrès.

**Le suivi quotidien est donc asynchrone :** chaque intervention significative est consignée dans `RustkinsActivity`, et les états/sprints sont mis à jour dans Azure Boards. Les autres membres de l'équipe peuvent ainsi connaître à tout moment : ce qui est en cours, terminé, bloqué, en pause, les US `Resolved`, les PR, les résultats de tests et les prochaines actions.

Le bot Discord peut produire des **rapports réguliers déterministes** à partir d'Azure Boards et des journaux d'activité, enrichis au besoin par les PR, CI et déploiements GitHub. Une synthèse IA facultative peut faciliter la lecture, sans inventer de faits. La présence d'un rapport ne signifie pas que les trois développeurs ont travaillé ce jour-là.

Un échange synchrone reste utile **à la demande**, par exemple pour lever une dépendance, décider d'une architecture, résoudre un blocage ou préparer une livraison ; il ne prend pas la forme d'un Daily imposé.

### 1.6. Sprint Review et Retrospective

**Sprint Review — validation des fonctionnalités et clôture des US :**

1. Réunir l'équipe (Timothy, Alexis et Loick) et consulter la liste des US `Resolved` du sprint.
2. Présenter les fonctionnalités sur la **plateforme de test**, avec les AC et preuves de tests.
3. Valider humainement chaque US conforme ; **la passer de `Resolved` à `Closed` pendant la review** et noter la décision.
4. Recueillir les remarques. Une US terminée et conforme reste `Closed` ; les améliorations demandées donnent lieu à de nouveaux work items, planifiés par Timothy au sprint suivant. Si un AC initial n'est pas respecté, l'US retourne à `Active`.
5. Dresser la liste des US non achevées et des reports.
6. Rédiger un **compte rendu de Sprint Review** : sprint/date/participants, objectif, US démontrées et validées avec leurs identifiants, décisions `Closed` ou report, remarques, Bugs/évolutions créés, liens vers preuves et actions suivantes.

**Sprint Retrospective — amélioration de la manière de travailler :**

1. Les trois développeurs discutent de ce qui a bien fonctionné et des difficultés (PR, CI, qualité des tests, blocages, organisation asynchrone, charges imprévues).
2. Choisir **quelques actions concrètes, attribuées et vérifiables** pour le sprint suivant, sans créer inutilement un grand nombre de Tasks.
3. Rédiger un **compte rendu de Sprint Retrospective** : sprint/date/participants, constats positifs, points à améliorer, décisions, actions, responsables et suivi au sprint suivant.

Les deux comptes rendus sont **obligatoires pour chaque sprint** et conservés dans `RustkinsDocs`, par exemple sous `docs/project-management/sprints/<sprint>/review.md` et `retrospective.md`. Ils peuvent être référencés dans Azure Boards et dans les rapports Discord. Les décisions collectives significatives peuvent donner lieu à une ADR séparée.

### 1.7. Fin de sprint et report des US

- **US `Closed` pendant la Sprint Review :** considérée comme terminée et comptabilisée parmi les livraisons du sprint.
- **US `Resolved` avant la review :** présentée pour validation ; si acceptée, passage à `Closed` **pendant la review**. Si un AC initial échoue, retour à `Active` puis report.
- **US non résolue à 100 % à la fin du sprint :** elle **n'est pas clôturée**. Elle reste `New` si aucun travail n'a commencé, `Active` si le développement se poursuit, ou `Blocked` / `On Hold` si l'interruption persiste ; tous ces états signifient **non terminée**. Timothy la **déplace au sprint suivant** sans forcer artificiellement son état à `Active`. Si le travail a déjà commencé, il peut rester dans l'état `Blocked` ou `On Hold` lorsque cette situation persiste : ces deux états demeurent **non terminés**, comme `Active`.
- **US finie mais nécessitant de nouveaux ajustements révélés en review :** si tous les AC initiaux et la DoD sont satisfaits, elle est **fermée**. Les demandes supplémentaires deviennent une **nouvelle US, une Task ou un Bug** du sprint suivant, selon leur nature et après arbitrage du backlog.
- **Tasks `Closed` rattachées à une US reportée :** elles restent fermées et historiquement liées à leur réalisation ; seules les Tasks inachevées poursuivent leur suivi.
- **Story Points** : ne comptabiliser comme livrés que les SP des US effectivement `Closed` après la review ; aucun crédit partiel pour une US inachevée.

Ne jamais clore artificiellement une US pour améliorer les indicateurs, ni modifier rétrospectivement les sprints pour masquer un report. Timothy documente les décisions de replanification.

---

