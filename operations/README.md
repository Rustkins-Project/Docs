# Procédures opérationnelles

Les runbooks décrivent **comment exécuter une opération** de manière reproductible : déploiement, maintenance, dépannage en production, réponse à incident, restauration.

- [Template runbook](../templates/runbook.md)
- Organisation progressive : \`deployment/\`, \`maintenance/\`, \`incident-response/\` selon les besoins.
- Inclure prérequis, commandes, vérifications, risques et procédure de retour arrière lorsque applicable.
- Éviter d'y stocker des secrets ou des adresses privées : fournir un mécanisme de résolution des paramètres de l'environnement.

Une note de résolution d'incident réutilisable peut appartenir à [knowledge/](../knowledge/README.md) ; la **procédure opératoire normative** appartient ici.
