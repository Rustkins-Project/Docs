---
id: DOC-0027
title: "Definition of Done, tests et preuves de validation"
type: project-management
status: approved
summary: "Conditions de clôture des activités, Tasks et US, politique des tests, preuves CI et validation en Sprint Review."
topics: [dod, testing, acceptance, ci]
applies_to: [all]
---

# Definition of Done, tests et preuves de validation

## 5. Definition of Done (DoD) — les trois niveaux

### 5.1. Activité RustkinsActivity

Une activité peut être considérée achevée lorsque :

- [ ] Son objectif, son auteur/profil, sa chronologie et son contexte sont consignés.
- [ ] Les résultats réels sont indiqués ; tests et PR sont référencés lorsqu'ils existent.
- [ ] Les blocages et actions restantes sont documentés.
- [ ] L'état final est cohérent (`completed` ou `abandoned`) et la date de fin est renseignée.
- [ ] Le work item Azure lié, s'il existe, est référencé ; **son état n'est pas supposé terminé**.

La clôture d'une conversation Codex/Claude **ne clôt pas automatiquement l'activité** : un agent peut interrompre, reprendre ou passer le relais.

### 5.2. Task Azure — `Closed`

La Task est `Closed` si et seulement si :

- [ ] Le résultat précis annoncé dans la description est implémenté ou produit.
- [ ] Les tests ou vérifications **pertinents pour cette Task** ont été exécutés et réussis, preuves disponibles.
- [ ] Les modifications sont intégrées dans la branche de référence appropriée **lorsqu'une intégration est requise** ; les PR nécessaires sont revues/approuvées selon la règle d'équipe.
- [ ] La documentation et/ou les contrats concernés sont à jour, si nécessaire.
- [ ] Aucun blocage ni sous-travail connu indispensable à sa réussite n'est laissé caché.
- [ ] L'état, les liens PR/CI et le compte rendu sont mis à jour dans Azure ; l'activité associée est tenue à jour.

**Les tests sont produits et exécutés pendant les Tasks de développement, sans créer de Tasks spécifiques aux tests.** Les E2E globaux de l'US ne sont pas tous exigés pour fermer chaque Task isolément, mais ils sont **obligatoires pour résoudre l'US**. Une Task ne peut pas être clôturée si sa propre implémentation est manifestement défectueuse.

### 5.3. User Story — porte `Active` → `Resolved`

Cette **DoD technique et fonctionnelle est obligatoire**. Les conditions sont **cumulatives** :

#### A. Fonctionnalité / acceptance criteria

- [ ] Tous les AC sont satisfaits et individuellement vérifiés ; l'équipe documente la preuve ou le scénario de test correspondant.
- [ ] Les cas limites et les règles métier identifiés sont couverts.
- [ ] Aucun Bug bloquant ou défaut connu invalidant un AC n'est ouvert.

#### B. Tests : trois niveaux obligatoires

- [ ] **Tests unitaires** : implémentés/actualisés pour le périmètre de l'US, exécutés et **réussis**.
- [ ] **Tests d'intégration** : implémentés/actualisés pour les composants et échanges concernés (DB, API, RabbitMQ, stockage, Docker, etc.), exécutés et **réussis**.
- [ ] **Tests E2E** : scénarios représentatifs couvrant le parcours externe complet de l'US (API, moteur, worker, interface selon le cas), implémentés/actualisés, exécutés et **réussis**.
- [ ] La **CI de la révision livrée** est verte pour les suites pertinentes ; preuves de runs et résultats consultables.
- [ ] Aucun test rouge ou désactivé sans justification ne masque un défaut concernant l'US.

**Règle retenue :** les trois catégories de tests sont exigées pour chaque US, intégrées au flux normal de réalisation, **sans Tasks de mise en place des tests**. Pour les US très techniques, définir un E2E technique observable (par exemple, déclencher un pipeline puis vérifier le statut, les sorties et les logs), plutôt que d'omettre la catégorie de test. Toute impossibilité exceptionnelle doit faire l'objet d'une **dérogation explicite et humaine, documentée**, et non d'une décision unilatérale de l'agent ; tant qu'elle n'est pas approuvée, l'US ne peut pas passer à `Resolved`.

#### C. Intégration, documentation, dépendances

- [ ] Toutes les **Tasks enfants sont dans l'état `Closed`** (aucune `New`, `Active`, `Blocked` ou `On Hold` restante ; un retrait de périmètre nécessite une décision humaine documentée).
- [ ] Code fusionné sur la branche de référence, revues requises obtenues, et artefact/build identifié.
- [ ] Documentation utilisateur, technique, configuration, migration et ADR pertinentes mises à jour si le changement le nécessite.
- [ ] Dépendances bloquantes levées ; aucun critère de fin repoussé discrètement hors périmètre.

#### D. Plateforme de test

- [ ] La **version contenant l'US est déployée sur la plateforme de test** partagée, accessible à l'équipe.
- [ ] Commit, build, image(s) et/ou release réellement déployés sont identifiables.
- [ ] Health checks, configuration et migrations nécessaires sont corrects.
- [ ] Les E2E et validations automatisées applicables ont été exécutés **contre la version déployée**, avec résultats verts.
- [ ] L'URL ou le chemin d'accès, le pipeline de déploiement et le run de tests sont référencés sur l'US.

**Transition autorisée :** `Active` → `Resolved` seulement quand **A + B + C + D** sont satisfaits. L'US est alors **prête à être validée**, mais **pas encore terminée**.

### 5.4. User Story — porte `Resolved` → `Closed` **pendant la Sprint Review**

La **Sprint Review est le moment officiel de clôture des US du sprint**. Chaque US `Resolved` est présentée à l'équipe **Timothy, Alexis et Loick** sur la plateforme de test. L'acceptation est une décision **humaine et traçable de l'équipe**, et non une décision automatique de l'agent.

Pour passer une US à `Closed` :

- [ ] L'US était réellement `Resolved` : tous ses AC et éléments de DoD sont déjà satisfaits, les tests réussis, toutes ses Tasks `Closed`, le déploiement de test prouvé.
- [ ] La fonctionnalité a été **présentée et validée lors de la Sprint Review** par les membres de l'équipe présents.
- [ ] La décision d'acceptation, la référence à l'US, les participants et les éventuelles remarques figurent dans le **compte rendu de Sprint Review**.
- [ ] Aucun défaut connu ne remet en cause les **AC initialement convenus** ou la DoD.
- [ ] La transition vers `Closed` est effectuée au terme de cette validation, sans clôture automatique à la fusion d'une PR ou à la fin d'une session IA.

**Demandes de modification à la suite de la review :** si l'US est effectivement terminée et que les AC initiaux sont satisfaits, elle est **clôturée**. Les demandes d'amélioration, adaptations ou ajouts recueillis pendant la review deviennent, selon leur nature, **une nouvelle US, une Task autonome ou un Bug à planifier au sprint suivant**. Ne pas rouvrir l'US acceptée simplement parce que l'équipe a identifié une évolution supplémentaire.

**En revanche, si la review démontre qu'un AC initial ou la DoD n'est pas respecté, l'US n'était pas réellement terminée :** elle retourne à `Active`, la correction est tracée, puis elle est reportée si nécessaire. Une évolution hors périmètre et un manquement à l'engagement initial ne doivent pas être confondus.

Les US `Resolved` **ne deviennent `Closed` que pendant la Sprint Review**. Les US non résolues à 100 % restent ouvertes et sont traitées selon la politique de fin de sprint ([§ 7.7](sprint-process.md)).

### 5.5. DoD complémentaires : Bug, Issue, Feature, Epic

| Type | Critères recommandés |
|---|---|
| Bug `Resolved` | Correction intégrée, tests de non-régression écrits/exécutés, correctif déployé en test si le défaut est reproductible sur cette plateforme |
| Bug `Closed` | Correction vérifiée et acceptée, régression absente sur la révision testée |
| Issue `Closed` | Investigation réalisée, livrable/recommandation consigné, résultat diffusé ; ADR si décision structurante approuvée |
| Feature `Closed` | Toutes les US nécessaires du regroupement sont `Closed`, périmètre cohérent, aucun manque masqué |
| Epic `Closed` | Features requises `Closed`, critères du lot satisfaits, release/version préparée et livraison prouvée, accord humain pour déclarer le lot livré |

---

## 6. Politique de tests, preuves et validation

### 6.1. Niveaux de test pour Rustkins

| Niveau | Finalité | Exemple Rustkins | Preuve attendue |
|---|---|---|---|
| Unitaire | Isoler une règle / fonction | Évaluation des dépendances et états `skipped` / `failed` dans le DAG | Lien CI vers les tests unitaires, liste/cas utiles |
| Intégration | Vérifier plusieurs composants et leurs contrats | Scheduler + PostgreSQL + RabbitMQ ; Worker + Docker ou RustFS dans un environnement de test contrôlé | Lien CI vers suite intégration, logs et rapports pertinents |
| E2E | Vérifier le comportement complet observable | Déclencher un run depuis l'API, voir les jobs exécutés, récupérer le statut et les logs | Lien E2E sur la plateforme de test, version déployée et scénario réussi |

Les tests sont développés et exécutés en continu au fil des Tasks : une Task exclusivement consacrée à la mise en place des tests n'est pas créée. Un test déclaré « écrit » mais non exécuté **ne vaut pas** preuve de réussite. Les tests doivent viser la même version de référence que celle déployée en test ; une suite verte sur un autre commit ne suffit pas à elle seule.

### 6.2. Traçabilité recommandée d'une US

Dans Azure Boards, conserver au minimum :

- les PR fusionnées et les commits/artefacts effectivement déployés ;
- les résultats CI des **tests unitaires, d'intégration et E2E**, et le résultat du déploiement test ;
- l'identité de l'environnement de test et la version effectivement présente ;
- la liste des AC vérifiés et leurs preuves ;
- le **compte rendu de Sprint Review** attestant la décision humaine de validation, avec les participants (Timothy, Alexis, Loick selon présence) ;
- les Bugs et Issues significatifs, ainsi que leurs résolutions ;
- les documents actualisés dans `RustkinsDocs`.

**Association PR ↔ work item :** Rustkins utilise **l'application Azure Boards pour GitHub** afin de lier les Pull Requests GitHub aux éléments de travail Azure. Les PR référencent les identifiants Azure via la syntaxe **`AB#123`** dans leur description (ou une autre liaison explicite reconnue par l'application). Le lien doit être vérifiable depuis Azure Boards ; une simple mention textuelle non reconnue n'est pas une preuve suffisante.

**Contrôle à venir :** une **CI GitHub** vérifiera qu'une PR porte au moins une référence Azure valide et liée aux work items concernés, avant son intégration. Une référence absente, erronée ou non accessible doit faire échouer le contrôle ; la vérification complète dépendra des permissions et de l'API Azure disponibles.

**Attention :** utiliser une référence simple **`AB#123`**, sans formule de fermeture automatique telle que **`Fixes AB#123`**. Les automatismes GitHub/Azure ne doivent **jamais clôturer une US à la fusion de sa PR** : cette clôture intervient uniquement après validation à la Sprint Review.

Pour faciliter l'automatisation des vérifications, consigner ces références dans une section conventionnelle **`## Preuves de validation`** dans l'US ou dans un commentaire de validation structuré, plutôt que de multiplier immédiatement les champs Azure personnalisés.

### 6.3. Cas particuliers

- **Tests rouges** : aucune transition vers `Resolved`, même si le développeur les estime non pertinents, sans analyse et décision explicite.
- **Tests E2E instables** : fiabiliser ou isoler la cause ; ne pas traiter un seul run vert aléatoire comme validation robuste.
- **Plateforme indisponible** : l'US reste `Active`, `Blocked` ou `On Hold` selon la cause réelle ; ne pas passer `Resolved` par anticipation.
- **Changement pendant la recette** : réexécuter les contrôles pertinents sur le nouvel artefact déployé.
- **Dérogation de tests** : accord humain explicite et traçable, durée/portée définies, dette suivie ; elle ne doit jamais être implicite.

---

