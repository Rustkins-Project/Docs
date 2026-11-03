---
id: DOC-0023
title: "Rustkins — Récapitulatif détaillé V0"
type: version
status: draft
summary: "Périmètre et fonctionnement de Rustkins V0 : moteur CI local, DAG, exécution Docker, persistance et interface."
topics: [V0, pipeline, docker, runtime]
applies_to: [V0]
---

# Rustkins — Récapitulatif détaillé V0

## 1. Objectif de la V0

La V0 de Rustkins correspond au **moteur CI local viable**.

L'objectif est de construire le socle minimum permettant d'exécuter un pipeline complet :

```txt
Dépôt Git public
  → déclenchement manuel
  → parsing du fichier YAML Rustkins
  → validation du DAG
  → création d'un run
  → exécution des jobs dans Docker
  → persistance des statuts
  → stockage des logs dans RustFS
  → affichage du résultat dans l'interface
```

À la fin de la V0, Rustkins doit être capable d'exécuter ce pipeline :

```yaml
jobs:
  build:
    image: node:22
    steps:
      - npm install
      - npm run build

  test:
    image: node:22
    needs: [build]
    steps:
      - npm test
```

La V0 doit rester volontairement limitée. L'objectif n'est pas encore de faire un concurrent complet de GitHub Actions ou GitLab CI, mais de prouver que le moteur interne fonctionne correctement.

---

## 2. Périmètre fonctionnel V0

### 2.1 Inclus

#### Gestion des projets

- Création d'un projet.
- Association d'un dépôt Git public.
- Définition du chemin du fichier YAML, par exemple `.rustkins/ci.yml`.
- Déclenchement manuel d'un pipeline.

#### Moteur de pipeline

- Parser YAML Rustkins.
- Validation du schéma YAML.
- Gestion de variables simples.
- Détection des erreurs de configuration.
- Détection des cycles dans le DAG.
- Construction du graphe d'exécution.
- Gestion des dépendances avec `needs`.

#### Exécution

- Clone Git au SHA demandé.
- Un conteneur Docker par job.
- Exécution séquentielle des steps d'un job.
- Capture des logs.
- Capture du code de retour.
- Gestion du fail-fast.
- Propagation des statuts `skipped` pour les jobs dépendants.

#### Persistance

- Historique des runs.
- Historique des jobs.
- Historique des steps.
- Persistance des logs dans RustFS.
- Référence aux logs stockée en base.
- Conservation des logs pendant 2 mois.

#### Interface utilisateur

- Liste des pipelines.
- Liste des runs.
- Détail d'un run.
- Affichage des statuts.
- Affichage des logs après exécution.

---

### 2.2 Hors périmètre V0

- Webhooks Git.
- Logs temps réel.
- Artifacts avancés.
- Cache.
- Secrets.
- Dépôts Git privés.
- Annulation des jobs.
- Scheduling avancé.
- Resource-aware scheduling.
- Services / sidecars.
- Docker Swarm multi-node.
- Kubernetes.
- Authentification complète.
- Autorisations fines.

---

## 3. Décisions structurantes V0

| Sujet | Décision V0 |
|---|---|
| Déploiement dev | Docker Compose local |
| Déploiement V0 cible | VM unique mononode |
| Orchestration prod V0 | Docker Compose ou Swarm mono-node |
| Exposition HTTP | Traefik devant UI/API |
| HTTPS | Hors V0, HTTP au début |
| Git | Dépôts publics uniquement |
| Secrets | Hors V0 |
| Stockage relationnel | PostgreSQL |
| Queue | RabbitMQ |
| Stockage logs | RustFS |
| Exécution jobs | Docker Engine local |
| Worker | Conteneur avec accès au Docker socket |
| Logs | Conservés 2 mois |
| RabbitMQ Management | Exposé temporairement en V0 |
| RustFS Console | Optionnelle, exposée seulement admin/dev |

---

## 4. Architecture logique

```mermaid
flowchart TD
  User[Utilisateur / Navigateur] --> Traefik[Traefik HTTP]

  Traefik --> UI[UI Rustkins]
  Traefik --> API[API Rustkins]

  UI -->|XHR /api/*| Traefik

  API --> PG[(PostgreSQL)]
  API --> MQ[(RabbitMQ)]
  API --> FS[(RustFS)]
  API --> Git[Git public externe]

  Scheduler[Scheduler] --> PG
  Scheduler --> MQ

  Worker[Worker container] --> MQ
  Worker --> PG
  Worker --> FS
  Worker --> Git
  Worker --> DockerSocket[/Docker socket/]

  DockerSocket --> DockerEngine[Docker Engine local]
  DockerEngine --> JobContainer[Conteneur de job]

  DockerEngine --> Registry[Docker Registry public]
  JobContainer --> Internet[Internet packages npm/dotnet/etc.]

  PG --> PGVolume[(Volume postgres_data)]
  FS --> FSVolume[(Volume rustfs_data)]

  Admin[Admin Dev/Ops] --> RabbitUI[RabbitMQ Management V0]
  RabbitUI --> MQ
```

---

## 5. Architecture infra V0 mononode

Pour la V0, l'infrastructure cible est volontairement simple : **une seule VM**.

```txt
VM Rustkins V0
├── Docker Engine
├── Traefik
├── UI Rustkins
├── API Rustkins
├── Scheduler
├── Worker
├── PostgreSQL
├── RabbitMQ
└── RustFS
```

```mermaid
flowchart TD
  Internet[Utilisateur / LAN] --> VM[VM Rustkins V0]

  subgraph VM[VM unique Rustkins]
    Docker[Docker Engine]

    Traefik[Traefik HTTP]
    UI[UI Rustkins]
    API[API Rustkins]
    Scheduler[Scheduler]
    Worker[Worker]
    PG[(PostgreSQL)]
    MQ[(RabbitMQ)]
    FS[(RustFS)]

    PGVolume[(Volume postgres_data)]
    FSVolume[(Volume rustfs_data)]

    Docker --> Traefik
    Docker --> UI
    Docker --> API
    Docker --> Scheduler
    Docker --> Worker
    Docker --> PG
    Docker --> MQ
    Docker --> FS

    Traefik --> UI
    Traefik --> API

    API --> PG
    API --> MQ
    API --> FS

    Scheduler --> PG
    Scheduler --> MQ

    Worker --> MQ
    Worker --> PG
    Worker --> FS
    Worker --> DockerSock[/var/run/docker.sock/]

    PG --> PGVolume
    FS --> FSVolume
  end

  Worker --> Git[Git public externe]
  Worker --> Registry[Docker Registry public]
  Worker --> Packages[Internet packages]
```

---

## 6. Dimensionnement recommandé de la VM V0

### Minimum raisonnable

```txt
CPU    : 4 vCPU
RAM    : 8 Go
Disque : 80 à 100 Go
OS     : Debian ou Ubuntu Server
```

### Confortable

```txt
CPU    : 6 à 8 vCPU
RAM    : 16 Go
Disque : 150 Go+
OS     : Debian ou Ubuntu Server
```

La consommation dépend surtout des jobs exécutés. Un simple `npm install` ou `npm run build` peut consommer beaucoup de CPU, RAM et disque.

---

## 7. Réseaux Docker

### 7.1 Réseau public

Le réseau public sert uniquement à exposer les services accessibles depuis l'extérieur.

```txt
rustkins-public
├── traefik
```

### 7.2 Réseau privé

Le réseau privé permet aux services internes de communiquer entre eux.

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

Traefik est connecté aux deux réseaux :

```txt
public  → reçoit les requêtes utilisateur
private → route vers UI/API
```

---

## 8. Ports et exposition

| Service | Port interne | Exposé V0 | Remarque |
|---|---:|---|---|
| Traefik | 80 | Oui | Point d'entrée HTTP |
| UI | 80 | Non direct | Accessible via Traefik |
| API | 8080 | Non direct | Accessible via Traefik `/api` |
| PostgreSQL | 5432 | Non | Interne uniquement |
| RabbitMQ AMQP | 5672 | Non | Interne uniquement |
| RabbitMQ Management | 15672 | Oui temporaire | Dev/Ops V0 |
| RustFS API | 9000 | Non | Interne uniquement |
| RustFS Console | 9001 | Optionnel | Dev/Ops V0 |
| Docker Engine | socket | Non | Monté uniquement dans Worker |

---

## 9. Contrat de communication entre services

### 9.1 UI → API

L'UI est chargée par le navigateur depuis Traefik.

Ensuite, le JavaScript de l'UI appelle l'API via l'URL publique :

```txt
GET /api/projects
GET /api/pipelines
POST /api/pipelines/{id}/runs
GET /api/runs/{id}
GET /api/runs/{id}/jobs/{jobId}/logs
```

Flux réel :

```txt
Navigateur
  → Traefik
  → API
```

---

### 9.2 API → PostgreSQL

L'API utilise PostgreSQL pour :

- créer les projets ;
- créer les pipelines ;
- créer les versions de pipeline ;
- créer les runs ;
- lire l'historique ;
- lire les statuts ;
- afficher les détails d'un run.

Accès requis : lecture/écriture.

---

### 9.3 API → RabbitMQ

L'API publie un message lorsqu'un utilisateur déclenche un run.

Exemple de message `PipelineRunRequested` :

```json
{
  "run_id": "uuid",
  "project_id": "uuid",
  "pipeline_id": "uuid",
  "git_sha": "abc123",
  "branch": "main"
}
```

---

### 9.4 Scheduler → PostgreSQL

Le Scheduler lit :

- les runs en attente ;
- les jobs en attente ;
- les dépendances `needs` ;
- les statuts des jobs terminés.

Le Scheduler écrit :

- les jobs prêts en statut `queued` ;
- les runs en statut `running`, `success` ou `failed` ;
- les jobs dépendants en `skipped` en cas de fail-fast.

---

### 9.5 Scheduler → RabbitMQ

Quand un job est prêt, le Scheduler publie un message `JobRunRequested`.

```json
{
  "run_id": "uuid",
  "job_run_id": "uuid"
}
```

---

### 9.6 Worker → RabbitMQ

Le Worker consomme les messages `JobRunRequested`.

Règles V0 recommandées :

```txt
- queue durable
- messages persistants
- ack manuel
- retry limité
- dead-letter queue pour erreurs techniques
```

---

### 9.7 Worker → PostgreSQL

Le Worker met à jour :

- le statut du job ;
- le statut des steps ;
- les timestamps `started_at` / `finished_at` ;
- les codes de retour ;
- la référence du fichier de logs dans RustFS.

---

### 9.8 Worker → RustFS

Le Worker écrit les logs dans RustFS.

Convention proposée :

```txt
runs/{pipeline_run_id}/jobs/{job_run_id}/logs.txt
```

La base PostgreSQL ne stocke pas le contenu complet des logs. Elle stocke uniquement la clé de l'objet RustFS.

---

### 9.9 Worker → Docker Engine

Le Worker ne lance pas les commandes directement dans son propre conteneur.

Il demande au Docker Engine local de créer un conteneur par job :

```txt
Worker
  → Docker socket
  → Docker Engine
  → Job container
```

Actions Docker nécessaires :

```txt
pull image
create container
start container
read logs
wait container
remove container
```

Dans Docker Compose, cela implique :

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

Point de sécurité important : un conteneur ayant accès au Docker socket a quasiment le contrôle de l'hôte Docker. C'est accepté pour la V0, mais ce n'est pas un modèle de sécurité suffisant pour une production avancée.

---

### 9.10 Worker → Git public

Le Worker doit pouvoir cloner le dépôt public :

```txt
git clone
git checkout <sha>
```

En V0 :

```txt
Dépôts publics uniquement.
Pas de token Git.
Pas de clé SSH.
Pas de secrets.
```

---

### 9.11 Job container → Internet

Les conteneurs de job doivent avoir un accès sortant à Internet pour :

```txt
npm install
dotnet restore
composer install
cargo fetch
pip install
etc.
```

---

## 10. Modèle de données V0

### 10.1 Principe

Le modèle sépare :

```txt
Définition
  → project
  → pipeline
  → pipeline_version

Exécution
  → pipeline_run
  → job_run
  → step_run

Stockage lourd
  → logs dans RustFS
```

Le YAML utilisé pour un run est figé dans `pipeline_versions`, afin que l'historique reste fiable même si le dépôt évolue après l'exécution.

---

### 10.2 MCD Mermaid

```mermaid
erDiagram
  PROJECT ||--o{ PIPELINE : owns
  PIPELINE ||--o{ PIPELINE_VERSION : has
  PROJECT ||--o{ PIPELINE_RUN : has
  PIPELINE ||--o{ PIPELINE_RUN : executes
  PIPELINE_VERSION ||--o{ PIPELINE_RUN : used_by

  PIPELINE_RUN ||--o{ JOB_RUN : contains
  JOB_RUN ||--o{ STEP_RUN : contains

  PROJECT {
    uuid id
    string name
    string git_repository_url
    string default_branch
    datetime created_at
    datetime updated_at
  }

  PIPELINE {
    uuid id
    uuid project_id
    string name
    string yaml_path
    datetime created_at
    datetime updated_at
  }

  PIPELINE_VERSION {
    uuid id
    uuid pipeline_id
    string git_sha
    text yaml_content
    string yaml_hash
    json parsed_definition_json
    string validation_status
    json validation_errors_json
    datetime created_at
  }

  PIPELINE_RUN {
    uuid id
    uuid project_id
    uuid pipeline_id
    uuid pipeline_version_id
    string git_sha
    string branch
    string status
    string trigger_type
    datetime started_at
    datetime finished_at
    datetime logs_expires_at
    datetime created_at
  }

  JOB_RUN {
    uuid id
    uuid pipeline_run_id
    string job_key
    string name
    string image
    string status
    json needs_json
    int exit_code
    string log_object_key
    boolean log_expired
    datetime started_at
    datetime finished_at
  }

  STEP_RUN {
    uuid id
    uuid job_run_id
    int step_index
    string command
    string status
    int exit_code
    datetime started_at
    datetime finished_at
  }
```

---

### 10.3 Tables

#### `projects`

Représente un dépôt Git suivi par Rustkins.

| Colonne | Type | Description |
|---|---|---|
| `id` | uuid | Identifiant du projet |
| `name` | text | Nom du projet |
| `git_repository_url` | text | URL du dépôt Git public |
| `default_branch` | text | Branche par défaut |
| `created_at` | timestamp | Date de création |
| `updated_at` | timestamp | Date de mise à jour |

---

#### `pipelines`

Représente une configuration logique de pipeline.

| Colonne | Type | Description |
|---|---|---|
| `id` | uuid | Identifiant du pipeline |
| `project_id` | uuid | Projet lié |
| `name` | text | Nom affiché |
| `yaml_path` | text | Chemin du YAML dans le dépôt |
| `created_at` | timestamp | Date de création |
| `updated_at` | timestamp | Date de mise à jour |

---

#### `pipeline_versions`

Représente une version figée du YAML utilisée par un run.

| Colonne | Type | Description |
|---|---|---|
| `id` | uuid | Identifiant de version |
| `pipeline_id` | uuid | Pipeline lié |
| `git_sha` | text | SHA du dépôt |
| `yaml_content` | text | Contenu YAML brut |
| `yaml_hash` | text | Hash du YAML |
| `parsed_definition_json` | jsonb | Définition parsée |
| `validation_status` | text | Résultat de validation |
| `validation_errors_json` | jsonb | Erreurs éventuelles |
| `created_at` | timestamp | Date de création |

---

#### `pipeline_runs`

Représente l'exécution complète d'un pipeline.

| Colonne | Type | Description |
|---|---|---|
| `id` | uuid | Identifiant du run |
| `project_id` | uuid | Projet lié |
| `pipeline_id` | uuid | Pipeline exécuté |
| `pipeline_version_id` | uuid | Version YAML utilisée |
| `git_sha` | text | SHA exécuté |
| `branch` | text | Branche source |
| `status` | text | Statut du run |
| `trigger_type` | text | Type de déclenchement |
| `started_at` | timestamp | Début d'exécution |
| `finished_at` | timestamp | Fin d'exécution |
| `logs_expires_at` | timestamp | Date d'expiration des logs |
| `created_at` | timestamp | Date de création |

---

#### `job_runs`

Représente l'exécution d'un job du pipeline.

| Colonne | Type | Description |
|---|---|---|
| `id` | uuid | Identifiant du job run |
| `pipeline_run_id` | uuid | Run parent |
| `job_key` | text | Clé du job dans le YAML |
| `name` | text | Nom affiché |
| `image` | text | Image Docker utilisée |
| `status` | text | Statut |
| `needs_json` | jsonb | Dépendances |
| `exit_code` | int | Code de retour global du job |
| `log_object_key` | text | Clé RustFS des logs |
| `log_expired` | boolean | Logs supprimés ou non |
| `started_at` | timestamp | Début |
| `finished_at` | timestamp | Fin |

---

#### `step_runs`

Représente l'exécution d'une commande dans un job.

| Colonne | Type | Description |
|---|---|---|
| `id` | uuid | Identifiant du step run |
| `job_run_id` | uuid | Job parent |
| `step_index` | int | Ordre du step |
| `command` | text | Commande exécutée |
| `status` | text | Statut |
| `exit_code` | int | Code de retour |
| `started_at` | timestamp | Début |
| `finished_at` | timestamp | Fin |

---

## 11. Statuts V0

Les mêmes statuts peuvent être utilisés pour `pipeline_runs`, `job_runs` et `step_runs`.

```txt
pending
queued
running
success
failed
skipped
canceled
```

Même si `canceled` est hors périmètre V0, le prévoir dès maintenant évite une migration inutile plus tard.

---

## 12. Flux complet d'exécution

```mermaid
sequenceDiagram
  actor User as Utilisateur
  participant UI as UI
  participant API as API Rustkins
  participant PG as PostgreSQL
  participant MQ as RabbitMQ
  participant Scheduler as Scheduler
  participant Worker as Worker
  participant FS as RustFS
  participant Docker as Docker Engine
  participant Job as Job Container
  participant Git as Git public

  User->>UI: Clique sur Run pipeline
  UI->>API: POST /api/pipelines/{id}/runs

  API->>Git: Récupère le YAML au SHA demandé
  API->>API: Parse YAML
  API->>API: Valide schéma + DAG
  API->>PG: Crée pipeline_version
  API->>PG: Crée pipeline_run, job_runs, step_runs
  API->>MQ: Publie PipelineRunRequested
  API-->>UI: Run créé

  Scheduler->>MQ: Consomme ou observe le run demandé
  Scheduler->>PG: Lit jobs pending + needs
  Scheduler->>PG: Marque build queued
  Scheduler->>MQ: Publie JobRunRequested(build)

  Worker->>MQ: Consomme JobRunRequested(build)
  Worker->>PG: Marque build running
  Worker->>Git: Clone repo au SHA
  Worker->>Docker: Pull image node:22
  Worker->>Docker: Create/start container
  Docker->>Job: Exécute npm install puis npm run build
  Worker->>FS: Écrit logs build
  Worker->>PG: Marque build success/failed
  Worker->>MQ: Ack message

  Scheduler->>PG: Détecte test prêt si build success
  Scheduler->>PG: Marque test queued
  Scheduler->>MQ: Publie JobRunRequested(test)

  Worker->>MQ: Consomme JobRunRequested(test)
  Worker->>PG: Marque test running
  Worker->>Docker: Create/start container
  Docker->>Job: Exécute npm test
  Worker->>FS: Écrit logs test
  Worker->>PG: Marque test success/failed

  Scheduler->>PG: Calcule statut final du run
  UI->>API: GET /api/runs/{id}
  API->>PG: Lit statuts
  API-->>UI: Retourne détail du run
```

---

## 13. Gestion du DAG et du fail-fast

### 13.1 Exemple de DAG

```mermaid
flowchart TD
  build[build]
  test[test]

  build --> test
```

### 13.2 Règles

- Un job sans `needs` peut être exécuté dès le début du run.
- Un job avec `needs` attend que tous ses parents soient en `success`.
- Si un parent est `failed`, le job dépendant passe en `skipped`.
- Si un job échoue, ses descendants non exécutables sont propagés en `skipped`.
- Le run est `failed` si au moins un job est `failed`.
- Le run est `success` si tous les jobs sont `success` ou correctement ignorés selon les règles V0.

---

## 14. Exemple Docker Compose dev

Ce compose est un exemple de départ. Les images applicatives Rustkins sont volontairement nommées de manière générique.

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
      - traefik.http.routers.rustkins-ui.entrypoints=web
      - traefik.http.services.rustkins-ui.loadbalancer.server.port=80
    networks:
      - rustkins-private

  api:
    image: rustkins/api:dev
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      RUSTFS_ENDPOINT: http://rustfs:9000
      RUSTFS_ACCESS_KEY: rustkins
      RUSTFS_SECRET_KEY: rustkins-password
      RUSTFS_BUCKET_LOGS: rustkins-logs
    labels:
      - traefik.enable=true
      - traefik.http.routers.rustkins-api.rule=PathPrefix(`/api`)
      - traefik.http.routers.rustkins-api.entrypoints=web
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
    depends_on:
      - postgres
      - rabbitmq
    networks:
      - rustkins-private

  worker:
    image: rustkins/worker:dev
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      RUSTFS_ENDPOINT: http://rustfs:9000
      RUSTFS_ACCESS_KEY: rustkins
      RUSTFS_SECRET_KEY: rustkins-password
      RUSTFS_BUCKET_LOGS: rustkins-logs
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - worker_workdir:/workdir
    depends_on:
      - postgres
      - rabbitmq
      - rustfs
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
  worker_workdir:

networks:
  rustkins-public:
  rustkins-private:
```

---

## 15. Exemple Swarm mono-node pour prod V0

La prod V0 peut être déployée sur une VM unique avec Docker Swarm initialisé en mono-node.

```bash
docker swarm init
```

L'intérêt de Swarm mono-node en V0 :

- se rapprocher d'une stack de déploiement plus propre ;
- utiliser `docker stack deploy` ;
- préparer les contraintes de placement futures ;
- garder une complexité faible tant qu'il n'y a qu'un seul node.

Attention : ce n'est pas encore du vrai multi-node.

---

### 15.1 Stack Swarm V0 exemple

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
          - node.role == manager

  ui:
    image: rustkins/ui:v0
    networks:
      - private
    deploy:
      replicas: 1
      labels:
        - traefik.enable=true
        - traefik.http.routers.rustkins-ui.rule=PathPrefix(`/`)
        - traefik.http.routers.rustkins-ui.entrypoints=web
        - traefik.http.services.rustkins-ui.loadbalancer.server.port=80

  api:
    image: rustkins/api:v0
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      RUSTFS_ENDPOINT: http://rustfs:9000
      RUSTFS_ACCESS_KEY: rustkins
      RUSTFS_SECRET_KEY: rustkins-password
      RUSTFS_BUCKET_LOGS: rustkins-logs
    networks:
      - private
    deploy:
      replicas: 1
      labels:
        - traefik.enable=true
        - traefik.http.routers.rustkins-api.rule=PathPrefix(`/api`)
        - traefik.http.routers.rustkins-api.entrypoints=web
        - traefik.http.services.rustkins-api.loadbalancer.server.port=8080

  scheduler:
    image: rustkins/scheduler:v0
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
    networks:
      - private
    deploy:
      replicas: 1

  worker:
    image: rustkins/worker:v0
    environment:
      DATABASE_URL: postgres://rustkins:rustkins@postgres:5432/rustkins
      RABBITMQ_URL: amqp://rustkins:rustkins@rabbitmq:5672
      RUSTFS_ENDPOINT: http://rustfs:9000
      RUSTFS_ACCESS_KEY: rustkins
      RUSTFS_SECRET_KEY: rustkins-password
      RUSTFS_BUCKET_LOGS: rustkins-logs
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - worker_workdir:/workdir
    networks:
      - private
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

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
          - node.role == manager

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: rustkins
      RABBITMQ_DEFAULT_PASS: rustkins
    ports:
      - target: 15672
        published: 15672
        protocol: tcp
        mode: host
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - private
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  rustfs:
    image: rustfs/rustfs:latest
    environment:
      RUSTFS_ACCESS_KEY: rustkins
      RUSTFS_SECRET_KEY: rustkins-password
      RUSTFS_CONSOLE_ENABLE: "true"
    ports:
      - target: 9001
        published: 9001
        protocol: tcp
        mode: host
    volumes:
      - rustfs_data:/data
    networks:
      - private
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

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
docker stack deploy -c stack.v0.yml rustkins
```

---

## 16. Différence Compose dev / Swarm prod V0

| Sujet | Compose dev | Swarm mono-node V0 |
|---|---|---|
| Commande | `docker compose up -d` | `docker stack deploy` |
| Cible | Machine dev | VM V0 |
| Réseau | bridge Compose | overlay Swarm |
| Secrets | Variables simples | Variables simples en V0 |
| Volumes | volumes Docker locaux | volumes Docker locaux du node |
| Scalabilité | Non prioritaire | Non utilisée en V0 |
| Worker | 1 conteneur | 1 service, 1 replica |
| Docker socket | Monté dans Worker | Monté dans Worker |
| Multi-node | Non | Non, volontairement |

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
RUSTKINS_PUBLIC_URL
```

### Scheduler

```txt
DATABASE_URL
RABBITMQ_URL
SCHEDULER_POLL_INTERVAL_SECONDS
```

### Worker

```txt
DATABASE_URL
RABBITMQ_URL
RUSTFS_ENDPOINT
RUSTFS_ACCESS_KEY
RUSTFS_SECRET_KEY
RUSTFS_BUCKET_LOGS
WORKER_WORKDIR
DOCKER_HOST
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

## 18. Rétention des logs

Décision V0 : les logs sont supprimés après 2 mois.

Proposition :

```txt
À la création d'un pipeline_run :
  logs_expires_at = created_at + 60 jours

Une tâche quotidienne :
  - sélectionne les job_runs dont pipeline_run.logs_expires_at < now()
  - supprime l'objet RustFS log_object_key
  - met job_runs.log_expired = true
```

Le run reste visible dans l'historique, mais les logs ne sont plus consultables.

---

## 19. Points de vigilance infra/dev

### 19.1 Docker socket

Le Worker a besoin du Docker socket pour créer les conteneurs de job.

Risque : accès très puissant à l'hôte Docker.

Accepté en V0, à revoir en V1/V2.

---

### 19.2 Accès Internet

Doivent avoir un accès sortant Internet :

```txt
- Worker : clone Git public + pull image Docker
- Job containers : téléchargement de packages
- Docker Engine : pull images Docker
```

---

### 19.3 Volumes persistants

Doivent être persistés :

```txt
postgres_data
rabbitmq_data
rustfs_data
```

Sans ces volumes, la suppression d'un conteneur peut provoquer une perte de données.

---

### 19.4 RabbitMQ Management exposé

Accepté temporairement en V0.

À durcir plus tard :

```txt
- accès via bastion
- accès via PAW
- non exposé publiquement
- authentification forte
- réseau management séparé
```

---

### 19.5 RustFS Console exposée

Optionnelle en V0.

Même logique que RabbitMQ Management : pratique en dev, à éviter en prod réelle.

---

### 19.6 Swarm mono-node

Swarm mono-node ne résout pas encore :

```txt
- haute disponibilité
- volumes distribués
- placement multi-node des jobs
- scaling réel des workers
```

C'est seulement une étape intermédiaire plus proche d'un déploiement structuré.

---

## 20. Critères de validation V0

La V0 est considérée comme validée si :

1. Un projet peut être créé avec un dépôt Git public.
2. Un pipeline peut être déclenché manuellement.
3. Le YAML est récupéré, parsé et validé.
4. Le DAG est construit.
5. Un cycle dans le DAG est détecté et bloque l'exécution.
6. Un run est créé en base.
7. Les jobs et steps sont créés en base.
8. Le Worker clone le dépôt au SHA demandé.
9. Le Worker lance un conteneur Docker par job.
10. Les steps sont exécutés dans l'ordre.
11. Les codes de retour sont capturés.
12. Les logs sont stockés dans RustFS.
13. Les statuts sont visibles dans l'UI.
14. Les logs sont visibles après exécution.
15. Un échec de `build` empêche `test` de tourner.
16. Les jobs dépendants sont marqués `skipped`.
17. Le run final passe en `success` ou `failed` correctement.
18. Les données PostgreSQL survivent à un redémarrage des conteneurs.
19. Les logs RustFS survivent à un redémarrage des conteneurs.
20. La stack peut être lancée sur une VM V0 mononode.

---

## 21. Résumé final

La V0 doit rester simple, mais solide.

Elle doit prouver les fondations suivantes :

```txt
- modèle de données fiable
- exécution Docker/job
- orchestration par DAG
- communication API/Scheduler/Worker
- persistance PostgreSQL
- logs RustFS
- queue RabbitMQ
- déploiement Docker Compose dev
- déploiement VM mononode V0
```

Ce qui est volontairement repoussé :

```txt
- secrets
- dépôts privés
- webhooks
- logs temps réel
- cache
- artifacts avancés
- services/sidecars
- annulation
- multi-node
- Kubernetes
```

La priorité est d'éviter un moteur trop ambitieux dès le départ, tout en gardant une architecture qui pourra évoluer proprement en V1, V2 et V3.
