# Connaissances collectives

Notes techniques **durables et réutilisables** : résolutions de problèmes, pièges Rust, Docker, PostgreSQL, RabbitMQ, patterns et bonnes pratiques observées.

## Gouvernance

**Aucune revue humaine obligatoire pour une connaissance.** L'auteur ou l'agent peut proposer une PR, la publier et la fusionner sans approbation d'un pair, sous réserve des droits GitHub et des contrôles techniques éventuels. Une PR reste recommandée pour la traçabilité ; la politique d'équipe n'impose pas une approbation. Ne pas transformer automatiquement une connaissance en règle d'architecture officielle.

- [Template connaissance](../templates/knowledge.md)
- [Schéma de métadonnées](../schemas/knowledge.schema.json)
- [Conventions complètes](../conventions/knowledge-management.md)

## Statuts

- `candidate` : observation documentée, non encore vérifiée.
- `verified` : résultat vérifié, avec reproduction ou preuve dans le document ; **la vérification n'implique pas une revue par un collègue**.
- `outdated` : information devenue obsolète, conservée comme historique.

## Organisation

Classer par **sujet**, pas par développeur : `rust/`, `docker/`, `database/`, `messaging/`, `security/`, `patterns/`, `troubleshooting/`. Créer uniquement les sous-dossiers utiles. Rechercher et enrichir une note existante avant d'en créer une nouvelle. Référencer, sans recopier, les informations de RustkinsActivity.
