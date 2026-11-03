---
id: DOC-0021
title: "Rustkins — Inventaire des features par coût d'implémentation"
type: version
status: draft
summary: "Inventaire des fonctionnalités du moteur CI/CD, regroupées selon leur coût estimé et leur complexité d'implémentation."
topics: [features, roadmap, complexity]
applies_to: [all]
---

# Rustkins — Inventaire des features par coût d'implémentation

> **Légende coût :**
> **Faible** = quelques jours, peu de pièges · **Moyen** = une à deux semaines, des cas limites · **Élevé** = gros morceau, source de bugs.

---

## Socle incompressible — sans ça, pas de moteur

| Feature | Coût | Ce que ça ajoute concrètement |
|---|---|---|
| Parser de ton format YAML | Moyen | Validation de schéma, gestion des erreurs lisibles, résolution de la hiérarchie env. C'est ta porte d'entrée, elle doit être robuste. |
| DAG in-degree + détection de cycle | Faible | L'algo est trivial (compteur). Le coût réel = le câbler sur un état **persistant**, pas juste en mémoire. |
| Steps séquentiels par job | Faible | Boucle ordonnée, arrêt au premier step rouge, capture exit code par step. |
| 1 container Docker/job + champ `image` | Faible | bollard fait le travail. Create → start → wait → logs → remove. |
| Limites CPU/RAM container | Faible | Passées à l'API Docker au create. Presque gratuit. |
| RabbitMQ (ack manuel, durable, retry, DLQ) | Moyen | La config est simple, la **logique de retry safe** (dédup) l'est moins. |
| Logs persistés sur rustfs | Faible | Append fichier par job-run. |
| SQLite + entité job-run | Faible/Moyen | Schéma propre (définition vs instance), transitions transactionnelles. |
| Clone Git au SHA | Faible | libgit2 ou shell out. Attention aux repos privés (clés de déploiement). |
| Fail-fast (job rouge → downstream `skipped`) | Faible | Tombe naturellement du DAG : tu ne décrémentes pas. |
| Statut de base dans l'UI | Moyen | Liste de runs, statut par job. Le minimum pour "voir" le système. |

---

## Coût moyen — utile, négociable

| Feature | Coût | Ce que ça ajoute concrètement |
|---|---|---|
| Logs temps réel (SSE tail) | Moyen | Reconnexion, multi-client, rattrapage du déjà-écrit à la connexion. Tu peux le **couper au MVP** (logs après coup) et l'ajouter ensuite. |
| Scheduling resource-aware (admission CPU/RAM) | Moyen | Compteur de capacité nœud, réservation/libération. Sans ça tu sur-souscris le nœud. |
| Annulation (kill in-flight) | Moyen | Message kill → tuer les containers en cours → drainer la ready-queue. Propagation pas triviale. |
| At-least-once + clé d'idempotence | Moyen | Dédup côté worker pour que le retry ne ré-exécute pas un job déjà fait. |
| Webhook trigger (HMAC) | Faible | Vérif signature. Petit mais obligatoire pour Git. |
| Auth UI (JWT) | Faible/Moyen | Login + middleware. |
| API tokens scopés (trigger machine) | Faible | Tokens par projet. |
| Cache (clé = hash fichiers déclarés) | Moyen | Store + clé + invalidation. Pas le câblage transit qui est dur, c'est la **cohérence** (cache pollué = builds faux). |
| Artifacts upload/download | Moyen | Transit via store (docker cp), pas de FS partagé. |
| Rétention/GC artifacts + cache | Moyen | Politique + job de nettoyage. Sans ça, disque plein en une semaine. |
| `when: branch` / `run_if: all_success` | Faible | Enums déclaratifs. Remplacent le monstre `if` pour presque rien. |
| `when: changed: [paths]` | Moyen | git diff sur le range. **Supprime le job `changes` entier** → excellent rapport. |
| Hiérarchie env + `vars` vs `secrets` | Faible/Moyen | Merge global→job→step avec override. |

---

## Coût élevé — les vrais morceaux

| Feature | Coût | Ce que ça ajoute concrètement |
|---|---|---|
| **Services/sidecars + healthcheck + réseau isolé** | Élevé | Orchestration multi-container : réseau dédié par job, attente du healthcheck avant les steps, teardown propre. C'est le plus gros bloc de la V1, et **ton e2e en dépend** (mariadb). |
| Secrets (chiffrés + injection + **redaction logs**) | Moyen/Élevé | Crypto au repos (age/libsodium), gestion de la clé maître, et surtout le filtre de redaction sur **tout** flux de log avant persistance ET SSE. |
| Authz fine ("voir les logs" = permission) | Moyen | Modèle de rôles, parce que les logs peuvent fuiter des secrets. |
| `docker build` rootless (Kaniko/Buildah) | Moyen/Élevé | Mode d'exécution différent (pas de dockerd), tes 5 jobs `docker-*`. |
| Crash recovery (état orchestration repris) | Élevé | Persister chaque transition pour reprendre après crash API. **Coupable** : "crash = runs en cours marqués `failed`" coûte zéro et reste honnête pour un projet d'école. |
| Matrix statique + fan-in "tout vert" | Moyen | Expansion cartésienne au parsing + nœud agrégé qui attend toutes les variantes. |
