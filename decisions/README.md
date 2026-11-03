# Décisions d'architecture (ADR)

Une ADR formalise **une décision et ses conséquences**, pas une simple découverte ou étude.

- Template : [adr.md](../templates/adr.md)
- Contrat des métadonnées : [adr.schema.json](../schemas/adr.schema.json)
- Règles complètes : [knowledge-management.md](../docs/conventions/knowledge-management.md)

## Gouvernance obligatoire

- Toute **nouvelle ADR ou modification de décision** passe par une Pull Request.
- **Au moins une approbation humaine d'un autre membre de l'équipe** est requise avant fusion (Timothy, Alexis ou Loick, distinct de l'auteur). L'agent IA ne peut pas approuver.
- Une ADR n'est applicable qu'après **fusion sur main et approbation**. L'approbation d'une réponse Discord seule ne publie pas une ADR.
- Statuts : `proposed`, `accepted`, `rejected`, `superseded`.
- Une décision remplacée reste consultable : nouvelle ADR + liens `supersedes`, sans effacement du passé.
- Le champ `applies_to` permet de distinguer les versions de Rustkins.

**Important :** le texte décrit la politique d'équipe. L'obligation de revue devra être rendue bloquante via les règles de protection GitHub appropriées ; ce changement de configuration n'est pas fait dans cette PR. Il n'existe pas encore d'ADR rétroactive créée automatiquement à partir des documents historiques.
