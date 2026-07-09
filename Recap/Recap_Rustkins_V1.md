# Rustkins — Récapitulatif détaillé V1

## 1. Objectif de la V1

La V1 de Rustkins correspond au **moteur CI utilisable en équipe**.

L'objectif est de transformer le socle V0 en une plateforme réellement exploitable par une petite équipe :

```txt
Push Git / déclenchement manuel / déclenchement API
  → authentification et autorisations projet
  → réception webhook sécurisée par HMAC
  → récupération du YAML Rustkins version 2
  → création d'une PipelineVersion immuable
  → création d'un PipelineRun
  → scheduling avec ressources CPU/RAM
  → dispatch durable via outbox PostgreSQL + RabbitMQ
  → exécution sur plusieurs workers Docker
  → logs persistés et diffusés en temps réel via SSE
  → artifacts stockés dans RustFS
  → affichage des statuts, logs et artifacts dans l'interface
```

À la fin de la V1, Rustkins doit être capable d'exécuter un pipeline comme celui-ci :

```yaml
version: 2

name: Backend CI

on:
  push:
    branches:
      - main
      - develop
  manual: {}
  api: {}

vars:
  NODE_ENV: test

image: node:22

jobs:
  build:
    resources:
      cpu: 1000
      memory: 2048
    steps:
      - name: Install dependencies
        run: npm ci
      - name: Build application
        run: npm run build
    artifacts:
      paths:
        - dist/**

  test:
    needs:
      - build
    steps:
      - name: Run tests
        run: npm test
      - name: Generate coverage
        run: npm run coverage
        continue_on_error: true

  cleanup:
    needs:
      - build
      - test
    run_if: always
    image: alpine:latest
    steps:
      - run: echo "cleanup"
```

La V1 ne cherche pas encore à couvrir les usages CI/CD complexes de production. Elle doit surtout rendre Rustkins fiable pour plusieurs utilisateurs, plusieurs projets, plusieurs workers et des pipelines déclenchés automatiquement.

---

## 2. Périmètre fonctionnel V1

### 2.1 Inclus

#### Gestion des utilisateurs et des projets

- Authentification JWT.
- Refresh tokens révocables.
- Gestion des utilisateurs.
- Gestion des membres d'un projet.
- Rôles projet : `owner`, `maintainer`, `developer`, `viewer`.
- Tokens d'API par projet.
- Scopes de tokens d'API.
- Autorisation des actions sensibles : run, cancel, logs, artifacts, webhooks.

#### Déclenchement

- Déclenchement manuel.
- Déclenchement API authentifié.
- Webhooks Git.
- Providers supportés : GitHub, GitLab, Gitea.
- Vérification HMAC.
- Déduplication des livraisons webhook.
- Déclenchement automatique sur événement `push`.
- Filtrage par branche.
- Support de plusieurs pipelines par projet.

#### Moteur de pipeline

- Support de la spécification YAML `version: 2`.
- Variables globales `vars`.
- Variables par job.
- Image Docker globale surchargeable par job.
- Ressources CPU/RAM par job.
- Jobs optionnels avec `optional`.
- Steps non bloquantes avec `continue_on_error`.
- Politique d'exécution `run_if`.
- Valeurs `run_if` : `all_success`, `always`, `on_failure`.
- Normalisation du YAML vers un modèle interne complet.
- Création de `PipelineVersion` immuables.
- Séparation stricte entre configuration et runtime.

#### Exécution

- Support de plusieurs workers.
- Workers stateless.
- Scheduling centralisé.
- Réservation CPU/RAM avant dispatch.
- Limites Docker CPU/RAM appliquées aux conteneurs.
- État `waiting_resources`.
- Annulation d'un run.
- Annulation d'un job.
- Retry limité des erreurs d'infrastructure.
- Dead Letter Queue.
- Idempotence côté worker.
- Gestion explicite des statuts `warning`.
- Heartbeat workers.
- Détection des workers offline.

#### RabbitMQ et robustesse runtime

- RabbitMQ utilisé comme transport uniquement.
- PostgreSQL reste la source de vérité.
- Transactional outbox pour les messages runtime.
- Outbox Publisher côté Scheduler.
- Messages RabbitMQ légers, contenant uniquement des identifiants.
- ACK manuel côté worker.
- Retry uniquement pour les erreurs d'infrastructure.
- Déduplication des messages côté worker.

#### Logs

- Logs temps réel via Server-Sent Events.
- Endpoint SSE par job.
- Reconnexion automatique.
- Reprise via `Last-Event-ID`.
- Chunks de logs transportés par RabbitMQ.
- Logs complets persistés dans RustFS.
- PostgreSQL stocke seulement les métadonnées de flux.
- Buffer mémoire court côté API.

#### Artifacts

- Déclaration d'artifacts dans le YAML.
- Upload d'artifacts en fin de job.
- Archive `tar.gz`.
- Checksum SHA-256.
- Stockage binaire dans RustFS.
- Métadonnées en PostgreSQL.
- Téléchargement via API et URL signée.
- Restauration automatique des artifacts des jobs listés dans `needs`.
- Politique de rétention par projet.
- Nettoyage des artifacts expirés.

#### Interface utilisateur

- Login/logout.
- Liste des projets accessibles.
- Liste des pipelines.
- Détail d'un pipeline.
- Liste des runs.
- Détail d'un run.
- Statuts en temps réel ou quasi temps réel.
- Logs temps réel.
- Téléchargement d'artifacts.
- Annulation de run/job selon permissions.
- Gestion minimale des membres projet.
- Gestion minimale des tokens d'API et webhooks.

---

### 2.2 Hors périmètre V1

- Kubernetes.
- Cache avancé.
- Secrets complets.
- Injection sécurisée de secrets dans les jobs.
- Redaction complète des secrets dans les logs.
- Services / sidecars.
- Matrix builds.
- Pull request triggers.
- Tag triggers.
- Conditions avancées `when`.
- Conditions par fichiers modifiés.
- Artifacts nommés.
- Téléchargement sélectif d'artifacts.
- Partage d'artifacts entre pipelines.
- Version comparison UI.
- Rollback de pipeline.
- Organisations et teams avancées.
- Permissions fines au-delà des rôles projet V1.
- Scheduling intelligent avec priorités, labels ou affinités.
- Haute disponibilité complète.

---

## 3. Décisions structurantes V1

| Sujet | Décision V1 |
|---|---|
| Objectif produit | Utilisation en petite équipe |
| Spécification pipeline | YAML `version: 2` |
| Authentification | JWT + refresh tokens |
| Autorisation | Rôles projet + scopes API tokens |
| Webhooks | GitHub, GitLab, Gitea |
| Sécurité webhooks | HMAC obligatoire |
| Git events | `push` uniquement |
| Source de vérité runtime | PostgreSQL |
| Transport async | RabbitMQ |
| Publication RabbitMQ | Transactional outbox |
| Outbox Publisher | Sous-composant du Scheduler |
| Logs temps réel | SSE |
| Stockage logs | RustFS |
| Stockage artifacts | RustFS |
| Téléchargement artifacts | API + URL signée |
| Exécution jobs | Docker Engine |
| Workers | Plusieurs workers possibles |
| Scheduling | Centralisé dans le Scheduler |
| Ressources | CPU milliCPU + mémoire MB |
| Sélection worker | Premier worker compatible |
| Retry | Erreurs d'infrastructure uniquement |
| Tentatives max | 3 |
| Déploiement dev | Docker Compose |
| Déploiement cible | Docker Swarm distribué simple |
| Kubernetes | Hors V1 |
| Secrets | Hors V1, sauf secrets webhook chiffrés |

---

## 4. Architecture logique

```mermaid
flowchart TD
  Browser[Utilisateur / Navigateur] --> Traefik[Traefik HTTP]
  GitProvider[GitHub / GitLab / Gitea] --> Traefik
  ExternalSystem[Système externe] --> Traefik

  Traefik --> UI[UI Rustkins]
  Traefik --> API[API Rustkins]

  UI -->|XHR /api/*| API
  UI -->|SSE logs| API

  API --> PG[(PostgreSQL)]
  API --> FS[(RustFS)]
  API --> Scheduler[Scheduler]

  API -->|webhook push| Scheduler
  Scheduler --> PG
  Scheduler --> Outbox[Outbox Publisher]
  Outbox --> MQ[(RabbitMQ)]

  MQ --> Worker1[Worker 1]
  MQ --> Worker2[Worker 2]
  MQ --> WorkerN[Worker N]

  Worker1 --> PG
  Worker1 --> FS
  Worker1 --> Git[Git externe]
  Worker1 --> Docker1[Docker Engine worker 1]

  Worker2 --> PG
  Worker2 --> FS
  Worker2 --> Git
  Worker2 --> Docker2[Docker Engine worker 2]

  WorkerN --> PG
  WorkerN --> FS
  WorkerN --> Git
  WorkerN --> DockerN[Docker Engine worker N]

  Docker1 --> JobContainer1[Conteneur job]
  Docker2 --> JobContainer2[Conteneur job]
  DockerN --> JobContainerN[Conteneur job]

  Worker1 -->|log chunks| MQ
  Worker2 -->|log chunks| MQ
  WorkerN -->|log chunks| MQ
  MQ -->|rustkins.logs| API

  PG --> PGVolume[(Volume postgres_data)]
  FS --> FSVolume[(Volume rustfs_data)]
```

---

## 5. Architecture infra V1 distribuée simple

La V1 peut encore fonctionner sur une seule machine en dev, mais la cible est un déploiement distribué simple avec plusieurs workers.

```txt
Cluster Rustkins V1
├── Manager / core node
│   ├── Traefik
│   ├── UI Rustkins
│   ├── API Rustkins
│   ├── Scheduler
│   ├── Outbox Publisher
│   ├── PostgreSQL
│   ├── RabbitMQ
│   └── RustFS
│
├── Worker node 1
│   ├── Worker Rustkins
│   └── Docker Engine
│
├── Worker node 2
│   ├── Worker Rustkins
│   └── Docker Engine
│
└── Worker node N
    ├── Worker Rustkins
    └── Docker Engine
```

```mermaid
flowchart TD
  User[Utilisateur / Git provider] --> Manager[Node manager Rustkins]

  subgraph Manager[Core node]
    Traefik[Traefik]
    UI[UI]
    API[API]
    Scheduler[Scheduler + Outbox Publisher]
    PG[(PostgreSQL)]
    MQ[(RabbitMQ)]
    FS[(RustFS)]
  end

  subgraph W1[Worker node 1]
    Worker1[Worker]
    Docker1[Docker Engine]
  end

  subgraph W2[Worker node 2]
    Worker2[Worker]
    Docker2[Docker Engine]
  end

  Traefik --> UI
  Traefik --> API
  API --> PG
  API --> Scheduler
  API --> FS
  Scheduler --> PG
  Scheduler --> MQ
  MQ --> Worker1
  MQ --> Worker2
  Worker1 --> PG
  Worker1 --> FS
  Worker1 --> Docker1
  Worker2 --> PG
  Worker2 --> FS
  Worker2 --> Docker2
```

---

## 6. Dimensionnement recommandé V1

### Minimum raisonnable pour une petite équipe

```txt
Core node
CPU    : 4 vCPU
RAM    : 8 à 16 Go
Disque : 150 Go+
OS     : Debian ou Ubuntu Server

Worker node
CPU    : 4 vCPU
RAM    : 8 Go
Disque : 100 Go+
OS     : Debian ou Ubuntu Server
```

### Confortable

```txt
Core node
CPU    : 6 à 8 vCPU
RAM    : 16 à 32 Go
Disque : 300 Go+

Workers
CPU    : 8 vCPU par worker
RAM    : 16 Go par worker
Disque : 150 Go+ par worker
```

Le dimensionnement dépend surtout :

- du nombre de jobs parallèles ;
- des images Docker utilisées ;
- du poids des artifacts ;
- du volume de logs ;
- des builds nécessitant beaucoup de RAM, comme `npm`, `rustc`, `dotnet` ou les tests E2E.

---

## 7. Réseaux Docker / Swarm

### 7.1 Réseau public

Le réseau public expose uniquement le point d'entrée HTTP.

```txt
rustkins-public
└── traefik
```

### 7.2 Réseau privé applicatif

Le réseau privé permet aux services Rustkins de communiquer entre eux.

```txt
rustkins-private
├── traefik
├── ui
├── api
├── scheduler
├── worker
├── postgres
├── rabbitmq
└── rustfs
```

### 7.3 Réseau worker

En V1, les workers peuvent être distribués sur plusieurs nodes.

Ils doivent pouvoir accéder à :

```txt
- PostgreSQL
- RabbitMQ
- RustFS
- Git externe
- Docker Engine local
- Internet sortant pour les containers de job
```

Traefik reste connecté aux réseaux public et privé :

```txt
public  → reçoit navigateur, API tokens et webhooks
private → route vers UI/API
```

---

## 8. Ports et exposition

| Service | Port interne | Exposé V1 | Remarque |
|---|---:|---|---|
| Traefik | 80 / 443 | Oui | Point d'entrée HTTP/HTTPS |
| UI | 80 | Non direct | Accessible via Traefik |
| API | 8080 | Non direct | Accessible via Traefik `/api` |
| PostgreSQL | 5432 | Non public | Accessible par services internes |
| RabbitMQ AMQP | 5672 | Non public | Accessible par Scheduler/API/Workers |
| RabbitMQ Management | 15672 | Admin uniquement | À protéger fortement |
| RustFS API | 9000 | Non public | Accessible par API/Workers |
| RustFS Console | 9001 | Admin uniquement | Optionnelle |
| Docker Engine | socket local | Non | Monté uniquement dans chaque Worker |

En V1, l'exposition directe de RabbitMQ Management et RustFS Console doit être évitée en production réelle. L'accès doit passer par un réseau admin, un bastion ou une règle d'accès restreinte.

---

## 9. Contrat de communication entre services

### 9.1 UI → API

L'UI appelle l'API via Traefik.

Exemples :

```txt
POST /api/auth/login
POST /api/auth/refresh
GET /api/projects
GET /api/projects/{projectId}/pipelines
POST /api/pipelines/{pipelineId}/runs
POST /api/runs/{runId}/cancel
GET /api/runs/{runId}
GET /api/jobs/{jobRunId}/logs/stream
GET /api/artifacts/{artifactId}
```

Les appels sont authentifiés par JWT, sauf routes explicitement publiques comme les webhooks.

---

### 9.2 Git provider → API

Endpoints publics :

```txt
POST /api/webhooks/github
POST /api/webhooks/gitlab
POST /api/webhooks/gitea
```

Règles :

- signature obligatoire ;
- validation HMAC ;
- comparaison en temps constant ;
- déduplication par identifiant de livraison ;
- normalisation vers un modèle Git interne ;
- création de run uniquement si le projet et le pipeline existent.

---

### 9.3 API → PostgreSQL

L'API lit et écrit :

- utilisateurs ;
- refresh tokens ;
- projets ;
- membres ;
- tokens d'API ;
- webhooks ;
- pipelines ;
- versions de pipeline ;
- runs ;
- logs metadata ;
- artifacts metadata.

PostgreSQL est la source de vérité.

---

### 9.4 API → Scheduler

L'API demande au Scheduler de créer ou traiter un run pour :

- déclenchement manuel ;
- déclenchement API ;
- webhook push validé.

Le Scheduler construit le DAG runtime et prend les décisions d'ordonnancement.

---

### 9.5 Scheduler → PostgreSQL

Le Scheduler :

- crée les `JobRun` et `StepRun` ;
- évalue le DAG ;
- réserve CPU/RAM ;
- passe les jobs en `queued` ;
- crée les `OutboxMessage` ;
- libère les ressources ;
- gère les retries ;
- marque les jobs en `dead_lettered` si nécessaire ;
- calcule le statut final du run.

---

### 9.6 Scheduler / Outbox Publisher → RabbitMQ

Le Scheduler ne publie pas directement les messages critiques.

Il écrit d'abord un `OutboxMessage` dans PostgreSQL dans la même transaction que le changement d'état.

L'Outbox Publisher publie ensuite vers RabbitMQ.

Exemple de message `job.ready` :

```json
{
  "message_id": "uuid",
  "pipeline_run_id": "uuid",
  "job_run_id": "uuid",
  "pipeline_version_job_id": "uuid",
  "project_id": "uuid",
  "attempt": 1,
  "idempotency_key": "run:job",
  "created_at": "2026-01-01T12:00:00Z"
}
```

---

### 9.7 Worker → RabbitMQ

Le Worker consomme les jobs prêts.

Queues principales :

```txt
rustkins.jobs.ready.q
rustkins.jobs.retry.q
rustkins.jobs.dlq
rustkins.control.cancel.q
rustkins.logs.sse.q
```

Règles :

- ACK manuel ;
- reload du `JobRun` depuis PostgreSQL avant exécution ;
- ACK des doublons sans exécution ;
- publication de chunks de logs vers `rustkins.logs`.

---

### 9.8 Worker → PostgreSQL

Le Worker met à jour :

- statut du job ;
- statut des steps ;
- code de retour ;
- timestamps ;
- tentative en cours ;
- détails d'erreur infrastructure ;
- metadata logs ;
- metadata artifacts.

Le Worker ne réserve pas et ne libère pas de ressources. Cette responsabilité reste au Scheduler.

---

### 9.9 Worker → RustFS

Le Worker écrit :

- logs persistants ;
- archives d'artifacts.

Convention possible :

```txt
runs/{pipeline_run_id}/jobs/{job_run_id}/logs.txt
runs/{pipeline_run_id}/jobs/{job_run_id}/artifacts/{artifact_id}.tar.gz
```

---

### 9.10 API → RustFS

L'API lit les logs historiques et génère les URLs signées pour les artifacts.

RustFS n'est pas exposé directement aux utilisateurs finaux.

---

### 9.11 Worker → Docker Engine

Chaque Worker contrôle le Docker Engine local de son node.

Actions nécessaires :

```txt
pull image
create container avec limites CPU/RAM
mount workspace
start container
stream stdout/stderr
wait container
stop container si annulation
remove container
```

Le Docker socket reste un point de vigilance majeur. Il est accepté en V1 pour garder l'exécution simple, mais il ne constitue pas une isolation forte.

---

## 10. Modèle de données V1

### 10.1 Principe

Le modèle V1 sépare quatre grands domaines :

```txt
Identité / sécurité
  → user
  → user_refresh_token
  → project_member
  → api_token

Configuration
  → project
  → webhook_config
  → pipeline
  → pipeline_version
  → pipeline_version_job
  → pipeline_version_step
  → pipeline_version_job_dependency

Exécution
  → pipeline_run
  → job_run
  → job_run_dependency
  → job_run_attempt
  → step_run
  → run_event
  → outbox_message
  → worker_node

Stockage lourd
  → job_log_stream metadata
  → artifact metadata
  → contenu binaire dans RustFS
```

La définition d'un pipeline est immuable. Le runtime référence cette définition au lieu de dupliquer les commandes dans chaque run.

---

### 10.2 MCD Mermaid simplifié

```mermaid
erDiagram
  USER ||--o{ USER_REFRESH_TOKEN : has
  USER ||--o{ PROJECT_MEMBER : belongs_to
  PROJECT ||--o{ PROJECT_MEMBER : has
  PROJECT ||--o{ API_TOKEN : owns
  PROJECT ||--o{ WEBHOOK_CONFIG : owns
  WEBHOOK_CONFIG ||--o{ WEBHOOK_DELIVERY : receives

  PROJECT ||--o{ PIPELINE : owns
  PIPELINE ||--o{ PIPELINE_VERSION : has
  PIPELINE_VERSION ||--o{ PIPELINE_VERSION_JOB : defines
  PIPELINE_VERSION_JOB ||--o{ PIPELINE_VERSION_STEP : contains
  PIPELINE_VERSION_JOB ||--o{ PIPELINE_VERSION_JOB_DEPENDENCY : has

  PIPELINE_VERSION ||--o{ PIPELINE_RUN : used_by
  PIPELINE_RUN ||--o{ JOB_RUN : contains
  JOB_RUN ||--o{ STEP_RUN : contains
  JOB_RUN ||--o{ JOB_RUN_ATTEMPT : has
  JOB_RUN ||--o| JOB_LOG_STREAM : writes
  JOB_RUN ||--o{ ARTIFACT : produces
  WORKER_NODE ||--o{ JOB_RUN : executes
  PIPELINE_RUN ||--o{ RUN_EVENT : has
  PIPELINE_RUN ||--o{ OUTBOX_MESSAGE : emits
```

---

### 10.3 Tables principales

#### `users`

Représente un utilisateur authentifié Rustkins.

| Colonne | Description |
|---|---|
| `id` | Identifiant utilisateur |
| `email` | Email de connexion |
| `username` | Nom affiché |
| `password_hash` | Hash du mot de passe |
| `role_global` | Rôle global éventuel |
| `is_active` | Compte actif ou non |
| `last_login_at` | Dernière connexion |

---

#### `user_refresh_tokens`

Représente une session révocable.

| Colonne | Description |
|---|---|
| `user_id` | Utilisateur lié |
| `token_hash` | Refresh token hashé |
| `expires_at` | Expiration |
| `revoked_at` | Révocation |
| `user_agent` | Contexte client |
| `ip_address` | Adresse IP observée |

---

#### `project_members`

Associe un utilisateur à un projet.

| Colonne | Description |
|---|---|
| `project_id` | Projet |
| `user_id` | Utilisateur |
| `role` | `owner`, `maintainer`, `developer`, `viewer` |

---

#### `api_tokens`

Permet l'automatisation par projet.

| Colonne | Description |
|---|---|
| `project_id` | Projet lié |
| `name` | Nom du token |
| `token_prefix` | Préfixe affichable |
| `token_hash` | Token hashé |
| `scopes` | Scopes autorisés |
| `revoked` | Révocation |
| `expires_at` | Expiration optionnelle |
| `last_used_at` | Dernière utilisation |

---

#### `webhook_configs`

Stocke la configuration webhook d'un projet.

| Colonne | Description |
|---|---|
| `project_id` | Projet lié |
| `provider` | GitHub, GitLab ou Gitea |
| `encrypted_secret` | Secret HMAC chiffré |
| `enabled` | Webhook actif |

---

#### `webhook_deliveries`

Historique des webhooks reçus.

| Colonne | Description |
|---|---|
| `external_delivery_id` | Identifiant provider |
| `event_type` | Type d'événement |
| `commit_sha` | Commit reçu |
| `branch` | Branche reçue |
| `status` | `received`, `validated`, `processed`, `completed`, `rejected` |
| `error_message` | Erreur éventuelle |

---

#### `pipeline_versions`

Snapshot immuable d'un YAML.

| Colonne | Description |
|---|---|
| `pipeline_id` | Pipeline logique |
| `git_sha` | SHA Git |
| `yaml_hash` | Hash du YAML |
| `yaml_content` | YAML brut |
| `validation_status` | `valid` ou `invalid` |
| `validation_errors_json` | Erreurs de validation |

Unicité recommandée :

```txt
pipeline_id + git_sha
```

---

#### `pipeline_version_jobs`

Définition immuable d'un job parsé.

| Colonne | Description |
|---|---|
| `job_key` | Identifiant YAML |
| `image` | Image résolue |
| `run_if` | Politique d'exécution |
| `optional` | Job non bloquant |
| `cpu_limit_milli` | CPU demandé |
| `memory_limit_mb` | Mémoire demandée |
| `timeout_seconds` | Timeout job |

---

#### `pipeline_version_steps`

Définition immuable d'une commande.

| Colonne | Description |
|---|---|
| `pipeline_version_job_id` | Job parent |
| `step_key` | Identifiant interne |
| `name` | Nom lisible |
| `command` | Commande |
| `order_index` | Ordre d'exécution |
| `continue_on_error` | Step non bloquante |

---

#### `pipeline_runs`

Exécution complète d'une version.

| Colonne | Description |
|---|---|
| `pipeline_version_id` | Version utilisée |
| `triggered_by_user_id` | Utilisateur déclencheur |
| `webhook_delivery_id` | Webhook déclencheur |
| `cancelled_by_user_id` | Utilisateur ayant annulé |
| `trigger_type` | `manual`, `api`, `webhook` |
| `status` | Statut runtime |
| `branch` | Branche |
| `commit_sha` | Commit exécuté |
| `commit_message` | Message du commit |
| `author_name` | Auteur |
| `author_email` | Email auteur |

---

#### `job_runs`

Exécution runtime d'un job.

| Colonne | Description |
|---|---|
| `pipeline_run_id` | Run parent |
| `pipeline_version_job_id` | Définition exécutée |
| `worker_node_id` | Worker assigné |
| `status` | Statut runtime |
| `idempotency_key` | Clé d'idempotence |
| `attempt_count` | Nombre de tentatives |
| `max_attempts` | Tentatives max |
| `exit_code` | Code de retour |

---

#### `worker_nodes`

Représente un worker d'exécution.

| Colonne | Description |
|---|---|
| `name` | Nom logique |
| `hostname` | Hôte |
| `status` | `online`, `offline`, etc. |
| `total_cpu_milli` | Capacité CPU |
| `total_memory_mb` | Capacité mémoire |
| `reserved_cpu_milli` | CPU réservé |
| `reserved_memory_mb` | Mémoire réservée |
| `last_heartbeat_at` | Dernier heartbeat |

---

#### `outbox_messages`

Message durable à publier vers RabbitMQ.

| Colonne | Description |
|---|---|
| `exchange` | Exchange RabbitMQ |
| `routing_key` | Routing key |
| `event_type` | Type d'événement |
| `payload_json` | Payload |
| `status` | `pending`, `publishing`, `published`, `failed` |
| `attempts` | Tentatives de publication |
| `next_attempt_at` | Prochaine tentative |
| `last_error` | Dernière erreur |
| `published_at` | Date de publication |

---

#### `artifacts`

Métadonnées d'un artifact.

| Colonne | Description |
|---|---|
| `pipeline_run_id` | Run parent |
| `job_run_id` | Job producteur |
| `archive_name` | Nom de l'archive |
| `storage_key` | Clé RustFS |
| `size_bytes` | Taille |
| `checksum_sha256` | Checksum |
| `expires_at` | Expiration |
| `deleted` | Archive supprimée |

---

## 11. Statuts V1

### 11.1 `pipeline_runs`

```txt
queued
running
success
warning
failed
cancelled
```

### 11.2 `job_runs`

```txt
waiting_dependencies
waiting_resources
queued
running
success
warning
failed
skipped
cancel_requested
cancelled
retrying
dead_lettered
```

### 11.3 `step_runs`

```txt
running
success
warning
failed
skipped
```

### 11.4 `outbox_messages`

```txt
pending
publishing
published
failed
```

### 11.5 `webhook_deliveries`

```txt
received
validated
processed
completed
rejected
```

Le statut `warning` est structurant en V1. Il signifie : problème réel, mais non bloquant pour le pipeline.

---

## 12. Flux complet d'exécution

```mermaid
sequenceDiagram
  actor Dev as Développeur
  participant Git as Git provider
  participant API as API Rustkins
  participant PG as PostgreSQL
  participant Scheduler as Scheduler
  participant Outbox as Outbox Publisher
  participant MQ as RabbitMQ
  participant Worker as Worker
  participant Docker as Docker Engine
  participant FS as RustFS
  participant UI as UI

  Dev->>Git: git push
  Git->>API: POST webhook
  API->>API: Vérifie HMAC
  API->>PG: Enregistre WebhookDelivery
  API->>Git: Lit YAML au commit SHA
  API->>API: Parse + valide YAML v2
  API->>PG: Crée ou réutilise PipelineVersion
  API->>Scheduler: Demande création PipelineRun

  Scheduler->>PG: Crée PipelineRun, JobRuns, StepRuns
  Scheduler->>PG: Évalue DAG
  Scheduler->>PG: Sélectionne worker compatible
  Scheduler->>PG: Réserve CPU/RAM
  Scheduler->>PG: JobRun = queued + OutboxMessage(job.ready)

  Outbox->>PG: Lit OutboxMessage pending
  Outbox->>MQ: Publie job.ready
  Outbox->>PG: Marque OutboxMessage published

  MQ->>Worker: Livre job.ready
  Worker->>PG: Recharge JobRun et vérifie idempotence
  Worker->>Git: Clone repo au SHA
  Worker->>FS: Restaure artifacts des dépendances
  Worker->>Docker: Lance conteneur avec limites CPU/RAM
  Docker-->>Worker: stdout/stderr + exit code
  Worker->>FS: Persiste logs
  Worker->>MQ: Publie chunks de logs
  MQ->>API: Transmet chunks de logs
  API->>UI: SSE logs temps réel
  Worker->>FS: Upload artifacts
  Worker->>PG: Met à jour JobRun/StepRuns/Artifacts + RunEvent(job_finished)
  Worker->>MQ: Publie job.finished pour réveiller le Scheduler
  Worker->>MQ: ACK du message job.ready

  MQ->>Scheduler: Livre job.finished
  Scheduler->>PG: Recharge le JobRun finalisé depuis PostgreSQL
  Scheduler->>PG: Libère ressources
  Scheduler->>PG: Débloque jobs dépendants
  Scheduler->>PG: Calcule statut final du PipelineRun
```

RabbitMQ peut envoyer un message métier `job.finished` au Scheduler pour éviter qu'il vérifie en permanence la base.

Ce message sert de **signal de réveil** : il dit au Scheduler qu'un job mérite d'être relu.

Le signal fiable reste l'état final écrit en PostgreSQL par le Worker. Le Scheduler ne doit donc pas faire confiance au payload RabbitMQ pour libérer les ressources ou débloquer le DAG ; il recharge toujours le `JobRun` depuis PostgreSQL.

L'ACK RabbitMQ du message `job.ready` garde son rôle transport : confirmer que l'ordre d'exécution reçu par le Worker a été traité et ne doit pas être redistribué.

---

## 13. Gestion du DAG, du warning et des ressources

### 13.1 Exemple de DAG

```mermaid
flowchart TD
  build[build]
  test[test]
  cleanup[cleanup]
  failure[failure-report]

  build --> test
  build --> cleanup
  test --> cleanup
  test --> failure
```

### 13.2 Règles `run_if`

| Valeur | Règle |
|---|---|
| `all_success` | Le job s'exécute si toutes ses dépendances sont `success` ou `warning` |
| `always` | Le job s'exécute dès que toutes ses dépendances sont finales |
| `on_failure` | Le job s'exécute si au moins une dépendance est `failed` ou `dead_lettered` |

Par défaut :

```yaml
run_if: all_success
```

### 13.3 Règles `warning`

- Une step avec `continue_on_error: true` qui échoue devient `warning`.
- Un job contenant seulement des erreurs non bloquantes devient `warning`.
- Un job `optional: true` avec une erreur métier devient `warning`.
- Un pipeline avec au moins un job `warning`, sans échec obligatoire, devient `warning`.
- Un `warning` ne déclenche pas `run_if: on_failure`.

### 13.4 Règles ressources

- Un job ne devient `queued` que si un worker compatible existe.
- Le Scheduler réserve CPU/RAM avant le dispatch RabbitMQ.
- Si aucun worker n'a assez de ressources temporairement, le job passe `waiting_resources`.
- Si le job demande plus que la capacité de tous les workers, il passe `failed`.
- Les ressources sont libérées uniquement par le Scheduler, après état final du job.

### 13.5 Fail-fast V1

Le fail-fast V0 devient une évaluation plus fine :

- un job obligatoire `failed` bloque les descendants en `all_success` ;
- les descendants incompatibles passent `skipped` ;
- les jobs `always` peuvent encore tourner ;
- les jobs `on_failure` peuvent gérer les erreurs ;
- les jobs `warning` restent non bloquants.

---

## 14. Exemple Docker Compose dev V1

Ce compose représente un environnement de développement V1 avec deux workers locaux.

```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - rustkins-public
      - rustkins-private

  ui:
    image: rustkins/ui:dev
    labels:
      - traefik.enable=true
      - traefik.http.routers.rustkins-ui.rule=PathPrefix(`/`)
      - traefik.http.services.rustkins-ui.loadbalancer.server.port=80
    networks:
      - rustkins-private

  api:
    image: rustkins/api:dev
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      RUSTFS_ENDPOINT: http://rustfs:9000
      JWT_ISSUER: rustkins
      JWT_AUDIENCE: rustkins-ui
      RUSTKINS_PUBLIC_URL: http://localhost
    labels:
      - traefik.enable=true
      - traefik.http.routers.rustkins-api.rule=PathPrefix(`/api`)
      - traefik.http.services.rustkins-api.loadbalancer.server.port=8080
    depends_on:
      - postgres
      - rabbitmq
      - rustfs
    networks:
      - rustkins-private

  scheduler:
    image: rustkins/scheduler:dev
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      SCHEDULER_POLL_INTERVAL_SECONDS: 2
    depends_on:
      - postgres
      - rabbitmq
    networks:
      - rustkins-private

  worker-1:
    image: rustkins/worker:dev
    environment:
      WORKER_NAME: worker-1
      WORKER_TOTAL_CPU_MILLI: 4000
      WORKER_TOTAL_MEMORY_MB: 8192
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      RUSTFS_ENDPOINT: http://rustfs:9000
      WORKER_WORKDIR: /workdir
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - worker_1_workdir:/workdir
    networks:
      - rustkins-private

  worker-2:
    image: rustkins/worker:dev
    environment:
      WORKER_NAME: worker-2
      WORKER_TOTAL_CPU_MILLI: 4000
      WORKER_TOTAL_MEMORY_MB: 8192
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      RUSTFS_ENDPOINT: http://rustfs:9000
      WORKER_WORKDIR: /workdir
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - worker_2_workdir:/workdir
    networks:
      - rustkins-private

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: rustkins
      POSTGRES_USER: rustkins
      POSTGRES_PASSWORD: rustkins
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - rustkins-private

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: rustkins
      RABBITMQ_DEFAULT_PASS: rustkins
    ports:
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - rustkins-private

  rustfs:
    image: rustfs/rustfs:latest
    environment:
      RUSTFS_ACCESS_KEY: rustkins
      RUSTFS_SECRET_KEY: rustkins-password
      RUSTFS_CONSOLE_ENABLE: "true"
    ports:
      - "9001:9001"
    volumes:
      - rustfs_data:/data
    networks:
      - rustkins-private

volumes:
  postgres_data:
  rabbitmq_data:
  rustfs_data:
  worker_1_workdir:
  worker_2_workdir:

networks:
  rustkins-public:
  rustkins-private:
```

---

## 15. Exemple Swarm distribué pour V1

La V1 vise un Swarm simple avec plusieurs workers.

```bash
docker swarm init
docker node update --label-add rustkins.role=core <core-node>
docker node update --label-add rustkins.role=worker <worker-node-1>
docker node update --label-add rustkins.role=worker <worker-node-2>
```

Principes :

- services core placés sur le node `core` ;
- workers placés sur les nodes `worker` ;
- volumes PostgreSQL/RabbitMQ/RustFS sur le core node ;
- Docker socket monté uniquement dans les workers ;
- Traefik expose l'UI et l'API ;
- RabbitMQ/RustFS/PostgreSQL restent privés.

### 15.1 Stack Swarm V1 exemple

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v3.0
    command:
      - --providers.swarm=true
      - --providers.swarm.exposedbydefault=false
      - --entrypoints.web.address=:80
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - public
      - private
    deploy:
      placement:
        constraints:
          - node.labels.rustkins.role == core

  ui:
    image: rustkins/ui:v1
    networks:
      - private
    deploy:
      replicas: 1
      labels:
        - traefik.enable=true
        - traefik.http.routers.rustkins-ui.rule=PathPrefix(`/`)
        - traefik.http.services.rustkins-ui.loadbalancer.server.port=80

  api:
    image: rustkins/api:v1
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      RUSTFS_ENDPOINT: http://rustfs:9000
      RUSTKINS_PUBLIC_URL: http://rustkins.example.local
    networks:
      - private
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.rustkins.role == core
      labels:
        - traefik.enable=true
        - traefik.http.routers.rustkins-api.rule=PathPrefix(`/api`)
        - traefik.http.services.rustkins-api.loadbalancer.server.port=8080

  scheduler:
    image: rustkins/scheduler:v1
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      SCHEDULER_POLL_INTERVAL_SECONDS: 2
    networks:
      - private
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.rustkins.role == core

  worker:
    image: rustkins/worker:v1
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      RUSTFS_ENDPOINT: http://rustfs:9000
      WORKER_TOTAL_CPU_MILLI: 4000
      WORKER_TOTAL_MEMORY_MB: 8192
      WORKER_WORKDIR: /workdir
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - worker_workdir:/workdir
    networks:
      - private
    deploy:
      mode: global
      placement:
        constraints:
          - node.labels.rustkins.role == worker

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: rustkins
      POSTGRES_USER: rustkins
      POSTGRES_PASSWORD: rustkins
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - private
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.rustkins.role == core

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: rustkins
      RABBITMQ_DEFAULT_PASS: rustkins
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - private
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.rustkins.role == core

  rustfs:
    image: rustfs/rustfs:latest
    environment:
      RUSTFS_ACCESS_KEY: rustkins
      RUSTFS_SECRET_KEY: rustkins-password
      RUSTFS_CONSOLE_ENABLE: "false"
    volumes:
      - rustfs_data:/data
    networks:
      - private
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.rustkins.role == core

volumes:
  postgres_data:
  rabbitmq_data:
  rustfs_data:
  worker_workdir:

networks:
  public:
    driver: overlay
  private:
    driver: overlay
```

Déploiement :

```bash
docker stack deploy -c stack.v1.yml rustkins
```

---

## 16. Différence Compose dev / Swarm V1

| Sujet | Compose dev V1 | Swarm V1 |
|---|---|---|
| Commande | `docker compose up -d` | `docker stack deploy` |
| Cible | Machine dev | Plusieurs nodes |
| Réseau | Bridge Compose | Overlay Swarm |
| Workers | Plusieurs conteneurs locaux | Un service global sur nodes workers |
| Placement | Non significatif | Labels `core` / `worker` |
| Volumes | Volumes locaux | Volumes locaux du node ciblé |
| PostgreSQL | Local compose | Core node |
| RabbitMQ | Local compose | Core node |
| RustFS | Local compose | Core node |
| Docker socket | Même host dev | Chaque worker node |
| Ressources | Simulées par workers locaux | Capacité par worker node |
| Logs SSE | Local | Via API centrale |
| Artifacts | RustFS local | RustFS core |
| HA | Non | Limitée, pas complète en V1 |
| Kubernetes | Non | Non |

---

## 17. Variables d'environnement à prévoir

### API

```txt
DATABASE_URL
RABBITMQ_URL
RUSTFS_ENDPOINT
RUSTFS_ACCESS_KEY
RUSTFS_SECRET_KEY
RUSTFS_BUCKET_LOGS
RUSTFS_BUCKET_ARTIFACTS
RUSTKINS_PUBLIC_URL
JWT_SECRET
JWT_ISSUER
JWT_AUDIENCE
JWT_ACCESS_TOKEN_TTL_MINUTES
JWT_REFRESH_TOKEN_TTL_DAYS
WEBHOOK_SECRET_ENCRYPTION_KEY
SIGNED_URL_TTL_SECONDS
```

### Scheduler

```txt
DATABASE_URL
RABBITMQ_URL
SCHEDULER_POLL_INTERVAL_SECONDS
SCHEDULER_WORKER_HEARTBEAT_TIMEOUT_SECONDS
OUTBOX_PUBLISH_INTERVAL_SECONDS
JOB_MAX_ATTEMPTS
JOB_RETRY_DELAY_1_SECONDS
JOB_RETRY_DELAY_2_SECONDS
JOB_RETRY_DELAY_3_SECONDS
CANCEL_TIMEOUT_SECONDS
```

### Worker

```txt
WORKER_NAME
WORKER_HOSTNAME
WORKER_TOTAL_CPU_MILLI
WORKER_TOTAL_MEMORY_MB
WORKER_WORKDIR
WORKER_HEARTBEAT_INTERVAL_SECONDS
DATABASE_URL
RABBITMQ_URL
RUSTFS_ENDPOINT
RUSTFS_ACCESS_KEY
RUSTFS_SECRET_KEY
RUSTFS_BUCKET_LOGS
RUSTFS_BUCKET_ARTIFACTS
DOCKER_HOST
DEFAULT_JOB_CPU_MILLI
DEFAULT_JOB_MEMORY_MB
DEFAULT_JOB_TIMEOUT_SECONDS
```

### PostgreSQL

```txt
POSTGRES_DB
POSTGRES_USER
POSTGRES_PASSWORD
```

### RabbitMQ

```txt
RABBITMQ_DEFAULT_USER
RABBITMQ_DEFAULT_PASS
```

### RustFS

```txt
RUSTFS_ACCESS_KEY
RUSTFS_SECRET_KEY
RUSTFS_CONSOLE_ENABLE
```

---

## 18. Rétention des logs et artifacts

### 18.1 Logs

Les logs restent persistés dans RustFS.

Décision recommandée V1 :

```txt
Rétention logs par défaut : 60 jours
```

Principe :

```txt
À la création d'un PipelineRun :
  logs_expires_at = created_at + 60 jours

Une tâche quotidienne :
  - sélectionne les JobLogStreams expirés
  - supprime les objets RustFS
  - conserve les métadonnées en base
```

### 18.2 Artifacts

Décision V1 :

```txt
Rétention artifacts par défaut : 30 jours
Taille max artifact : 100 MB
Taille max artifacts par run : 500 MB
```

Chaque projet peut définir :

- rétention par défaut ;
- rétention maximale ;
- taille maximale d'un artifact ;
- taille maximale des artifacts d'un run.

Quand un artifact expire :

```txt
artifact.deleted = true
artifact.deleted_at = now()
objet RustFS supprimé
métadonnées conservées
```

Le run reste consultable, mais l'archive n'est plus téléchargeable.

---

## 19. Points de vigilance infra/dev

### 19.1 Docker socket

Chaque Worker ayant accès au Docker socket contrôle fortement son host Docker.

Accepté en V1, mais à revoir plus tard avec :

```txt
- sandbox plus stricte
- exécution rootless
- isolation par runtime dédié
- backend Kubernetes
```

---

### 19.2 Outbox obligatoire

RabbitMQ ne doit pas être traité comme source de vérité.

Toute décision runtime critique doit être persistée dans PostgreSQL avant publication RabbitMQ.

Sans outbox, une panne RabbitMQ au mauvais moment peut créer un run bloqué ou un job perdu.

---

### 19.3 Idempotence worker

RabbitMQ fonctionne en livraison at-least-once.

Le Worker doit toujours :

```txt
1. recharger JobRun depuis PostgreSQL
2. vérifier le statut courant
3. ignorer les jobs déjà terminés
4. ACK le doublon
```

Un même `JobRun` ne doit jamais être exécuté deux fois.

---

### 19.4 Logs temps réel

RabbitMQ transporte les chunks live, mais RustFS reste la source persistante.

Ordre recommandé côté Worker :

```txt
1. écrire le chunk dans RustFS
2. publier le chunk dans RabbitMQ
```

Si l'API ou RabbitMQ tombe, les logs doivent rester récupérables depuis RustFS.

---

### 19.5 Artifacts

Les artifacts doivent être validés strictement :

```txt
- pas de path traversal
- pas de chemins absolus
- pas de symlinks cassés
- checksum SHA-256
- extraction relative au workspace
```

La restauration automatique des artifacts de `needs` doit éviter les collisions de chemins.

---

### 19.6 Ressources workers

La réservation CPU/RAM doit être atomique.

Séquence obligatoire :

```txt
lock worker row
calcul capacité disponible
réservation CPU/RAM
JobRun = queued
création OutboxMessage
commit
```

Sinon deux jobs concurrents peuvent être assignés au même worker au-delà de sa capacité.

---

### 19.7 Webhooks

Un webhook ne doit jamais créer automatiquement un projet ou un pipeline.

La configuration Rustkins reste explicite :

```txt
projet existant
pipeline existant
yaml_path configuré
webhook actif
signature valide
branche compatible
```

---

### 19.8 Secrets

Les secrets complets sont hors V1.

Il faut donc éviter de présenter `vars` comme un mécanisme sûr pour :

```txt
- mots de passe
- tokens
- clés API
- clés privées
- credentials Docker
```

Seuls les secrets webhook sont stockés chiffrés, car le HMAC nécessite de relire la valeur originale.

---

## 20. Critères de validation V1

La V1 est considérée comme validée si :

1. Un utilisateur peut se connecter avec JWT.
2. Un refresh token permet de renouveler un access token.
3. Un utilisateur peut être membre d'un projet avec un rôle.
4. Les rôles bloquent correctement les actions non autorisées.
5. Un token d'API projet peut déclencher un pipeline avec le bon scope.
6. Un webhook GitHub, GitLab ou Gitea signé déclenche un run.
7. Un webhook avec signature absente ou invalide est rejeté en `401`.
8. Une livraison webhook dupliquée est ignorée.
9. Un push sur une branche non autorisée ne crée pas de run.
10. Un projet peut contenir plusieurs pipelines.
11. Le YAML `version: 2` est parsé et validé.
12. Les variables globales et de job sont fusionnées correctement.
13. L'image globale est utilisée quand le job n'a pas d'image.
14. Les ressources CPU/RAM par job sont normalisées.
15. Une `PipelineVersion` immuable est créée ou réutilisée par SHA.
16. Le modèle sépare configuration et runtime.
17. Le Scheduler réserve les ressources avant dispatch.
18. Un job sans worker compatible temporaire passe `waiting_resources`.
19. Un job trop gros pour tous les workers passe `failed`.
20. Un message `job.ready` est créé via `OutboxMessage`.
21. L'Outbox Publisher publie vers RabbitMQ après commit.
22. Un Worker recharge le `JobRun` depuis PostgreSQL avant exécution.
23. Un doublon RabbitMQ n'exécute pas deux fois le même job.
24. Une erreur infrastructure est retry avec les délais V1.
25. Après 3 tentatives, un job passe `dead_lettered`.
26. Une erreur métier ne déclenche pas de retry.
27. Une step `continue_on_error` en échec devient `warning`.
28. Un job `optional` en échec métier devient `warning`.
29. Un pipeline avec warning mais sans échec obligatoire devient `warning`.
30. `run_if: always` s'exécute après une dépendance échouée.
31. `run_if: on_failure` s'exécute seulement après `failed` ou `dead_lettered`.
32. Un run peut être annulé.
33. Un job en cours peut recevoir une demande d'annulation.
34. Les ressources sont libérées après état final.
35. Un Worker offline est détecté par heartbeat.
36. Les logs sont visibles en temps réel via SSE.
37. Une reconnexion SSE reprend les logs manquants.
38. Les logs complets restent consultables depuis RustFS.
39. Un job peut uploader un artifact.
40. Un artifact est archivé en `tar.gz`.
41. Le checksum SHA-256 est stocké.
42. Un artifact est téléchargeable via API et URL signée.
43. Un job dépendant restaure les artifacts directs de ses `needs`.
44. Les artifacts expirés sont supprimés de RustFS.
45. La stack peut tourner en Docker Compose dev.
46. La stack peut être déployée en Swarm simple avec plusieurs workers.

---

## 21. Résumé final

La V1 doit faire passer Rustkins d'un moteur local viable à un outil CI crédible pour une petite équipe.

Elle doit prouver les fondations suivantes :

```txt
- authentification JWT
- rôles projet
- tokens d'API scopés
- webhooks Git sécurisés
- YAML version 2
- PipelineVersion immuable
- modèle configuration/runtime séparé
- transactional outbox
- RabbitMQ comme transport uniquement
- scheduler centralisé
- workers multiples
- réservation CPU/RAM
- retry infrastructure
- idempotence worker
- annulation
- logs temps réel SSE
- logs persistés RustFS
- artifacts RustFS
- URL signées
- déploiement Docker Compose dev
- déploiement Swarm distribué simple
```

Ce qui reste volontairement repoussé :

```txt
- Kubernetes
- secrets complets
- cache
- services/sidecars
- matrix builds
- pull request triggers
- tag triggers
- conditions avancées when
- permissions très fines
- scheduling intelligent
- haute disponibilité complète
```

La priorité de la V1 est la fiabilité d'équipe : déclenchements automatiques, sécurité minimale solide, exécution distribuée simple, logs temps réel, artifacts et robustesse face aux doublons ou aux pannes temporaires.
