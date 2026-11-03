---
id: DOC-0019
title: "Rustkins Pipeline Specification v3 (V2 Scope)"
type: specification
status: draft
summary: "Format YAML v3 pour Rustkins V2 : secrets, conditions when, matrix, services, cache et environnements de déploiement."
topics: [pipeline, yaml, secrets, matrix, cache]
applies_to: [V2]
specification_version: "3"
---

# Rustkins Pipeline Specification v3 (V2 Scope)

## 1. Introduction

Ce document décrit la version 3 de la spécification YAML des pipelines Rustkins.

Cette version correspond au périmètre fonctionnel de la V2 du produit Rustkins.

La version 3 étend la version 2 avec les fonctionnalités nécessaires à une plateforme CI/CD avancée :

* secrets référencés et injectés de manière sécurisée ;
* conditions d'exécution avancées avec `when` ;
* matrix builds statiques ;
* fan-in automatique après une matrix ;
* services / sidecars ;
* réseaux isolés par job ;
* healthchecks ;
* cache déclaratif et immuable ;
* artifacts nommés ;
* téléchargement sélectif des artifacts ;
* environnements de déploiement ;
* déclenchements sur tags ;
* déclenchements sur pull requests ;
* inputs pour les déclenchements manuels et API.

Les expressions booléennes libres, les volumes hôte dans les services et un DSL de déploiement spécifique à Kubernetes ou Docker Swarm restent hors périmètre de cette version.

---

## 2. Compatibilité avec les versions précédentes

Un pipeline utilisant cette spécification doit déclarer :

```yaml
version: 3
```

La version 3 conserve les concepts des versions précédentes :

* `name`
* `image`
* `on`
* `vars`
* `jobs`
* `needs`
* `run_if`
* `optional`
* `resources`
* `timeout`
* `steps`
* `continue_on_error`
* `artifacts`

Les pipelines déclarés avec `version: 1` ou `version: 2` restent valides selon les règles de leur propre version.

Une fonctionnalité introduite en version 3 ne peut pas être utilisée dans un pipeline déclarant une version antérieure.

Exemple d'erreur :

```text
Field "services" requires pipeline version 3.
```

---

## 3. Emplacement du fichier

Les pipelines Rustkins sont stockés dans le dossier :

```text
.rustkins/
```

Chaque pipeline doit être défini dans un fichier YAML situé sous ce dossier.

Le nom du fichier est libre.

Exemples :

```text
.rustkins/ci.yml
.rustkins/backend.yml
.rustkins/release.yml
```

Plusieurs pipelines peuvent coexister dans un même dépôt.

---

## 4. Exemple complet

```yaml
version: 3

name: Application CI/CD

image: node:22

on:
  push:
    branches:
      include:
        - main
        - develop
      exclude:
        - experimental/*

    tags:
      include:
        - v*

  pull_request:
    branches:
      include:
        - main

    actions:
      - opened
      - synchronize
      - reopened

    checkout: merge

  manual:
    inputs:
      target:
        type: choice
        required: true
        options:
          - staging
          - production

      dry_run:
        type: boolean
        required: false
        default: false

  api:
    inputs:
      target:
        type: string
        required: true

vars:
  NODE_ENV: test

jobs:
  test:
    name: Tests Node ${{ matrix.node }} / ${{ matrix.database }}

    strategy:
      matrix:
        node:
          - "20"
          - "22"

        database:
          - postgres
          - mariadb

        exclude:
          - node: "20"
            database: mariadb

        include:
          - node: "23"
            database: postgres
            experimental: "true"

    image: node:${{ matrix.node }}

    vars:
      DATABASE_ENGINE: ${{ matrix.database }}

    cache:
      - name: npm-downloads
        scope: branch

        paths:
          - .npm/

        key:
          prefix: npm-node-${{ matrix.node }}
          files:
            - package-lock.json

        restore:
          prefixes:
            - npm-node-${{ matrix.node }}
            - npm

    services:
      database:
        image: postgres:16

        vars:
          POSTGRES_DB: application
          POSTGRES_USER: application

        secrets:
          POSTGRES_PASSWORD: postgres-test-password

        healthcheck:
          command: pg_isready -U application
          interval: 5
          timeout: 3
          retries: 10

    steps:
      - name: Install dependencies
        run: npm ci --cache .npm

      - name: Run tests
        run: npm test

    artifacts:
      - name: test-reports
        paths:
          - reports/**
        when: always
        retention: 7

  build:
    needs:
      - test

    when:
      changed:
        include:
          - src/**
          - package.json
          - package-lock.json

    steps:
      - run: npm ci
      - run: npm run build

    artifacts:
      - name: application
        paths:
          - dist/**
        when: success
        retention: 30

  deploy:
    needs:
      - build

    when:
      branches:
        include:
          - main

    environment:
      name: staging
      url: https://staging.example.com

    image: alpine/helm:latest

    secrets:
      KUBECONFIG:
        from: staging-kubeconfig
        target:
          type: file
          path: /run/secrets/kubeconfig

    artifact_download:
      auto: false

      items:
        - from: build
          name: application
          path: ./release

    steps:
      - name: Deploy application
        run: |
          helm upgrade --install application ./helm \
            --kubeconfig /run/secrets/kubeconfig
```

---

## 5. Structure globale

Un fichier pipeline version 3 peut contenir les champs suivants :

```yaml
version: integer
name: string
image: string
on: object
vars: object
secrets: object
jobs: object
```

### 5.1 Champs racine

| Champ | Type | Obligatoire | Description |
|---|---|---:|---|
| `version` | integer | Oui | Version de la spécification. |
| `name` | string | Oui | Nom lisible du pipeline. |
| `image` | string | Non | Image Docker par défaut des jobs. |
| `on` | object | Oui | Déclencheurs du pipeline. |
| `vars` | object | Non | Variables globales non sensibles. |
| `secrets` | object | Non | Secrets demandés globalement. |
| `jobs` | object | Oui | Définition des jobs. |

---

## 6. Version

```yaml
version: 3
```

### Contraintes

* obligatoire ;
* entier ;
* doit valoir `3` pour utiliser les fonctionnalités de cette spécification.

Erreurs possibles :

```text
Pipeline version is missing.
Unsupported pipeline version: 4.
Field "matrix" requires pipeline version 3.
```

---

## 7. Nom du pipeline

```yaml
name: Application CI/CD
```

### Contraintes

* obligatoire ;
* chaîne non vide ;
* longueur maximale recommandée : 100 caractères.

---

## 8. Image globale

Une image Docker peut être définie à la racine :

```yaml
image: node:22
```

Elle est utilisée par tous les jobs qui ne déclarent pas leur propre image.

```yaml
image: node:22

jobs:
  build:
    steps:
      - run: npm run build

  security:
    image: aquasec/trivy:latest
    steps:
      - run: trivy fs .
```

### Contraintes

* optionnel ;
* chaîne non vide ;
* chaque job doit disposer d'une image après résolution ;
* l'image d'un job surcharge l'image globale ;
* l'interpolation matrix est autorisée au niveau d'un job.

---

## 9. Déclencheurs

Le champ `on` définit les événements pouvant démarrer le pipeline.

La version 3 supporte :

* `push`
* `pull_request`
* `manual`
* `api`

Un pipeline doit déclarer au moins un trigger.

---

## 9.1 Trigger `push`

Le trigger `push` peut filtrer les branches et les tags.

```yaml
on:
  push:
    branches:
      include:
        - main
        - develop
      exclude:
        - experimental/*

    tags:
      include:
        - v*
      exclude:
        - v*-alpha
```

### Branches

```yaml
branches:
  include:
    - main
    - release/*
  exclude:
    - experimental/*
```

### Tags

```yaml
tags:
  include:
    - v*
  exclude:
    - v*-alpha
```

### Règles

* `include` et `exclude` sont optionnels ;
* chaque liste doit contenir des chaînes non vides ;
* les patterns glob simples sont autorisés ;
* une branche ou un tag exclu ne déclenche pas le pipeline ;
* si `branches` et `tags` sont tous les deux absents, tous les push compatibles avec le provider peuvent déclencher le pipeline ;
* un événement Git sur une branche est évalué contre `branches` ;
* un événement Git sur un tag est évalué contre `tags`.

---

## 9.2 Trigger `pull_request`

```yaml
on:
  pull_request:
    branches:
      include:
        - main

    actions:
      - opened
      - synchronize
      - reopened

    checkout: merge
```

Le filtre `branches` cible la branche de destination de la pull request.

### Actions supportées

```text
opened
synchronize
reopened
```

Des actions supplémentaires pourront être ajoutées dans une version future.

### Mode de checkout

| Valeur | Description |
|---|---|
| `head` | Exécute le dernier SHA de la branche source. |
| `merge` | Exécute un merge temporaire de la branche source avec la branche cible. |

Valeur par défaut :

```yaml
checkout: merge
```

### Conflit de merge

Si `checkout: merge` est utilisé et que le merge temporaire échoue :

```text
PipelineRun = failed
error_type = merge_conflict
```

Aucun job n'est exécuté.

---

## 9.3 Sécurité des pull requests

Une pull request peut provenir :

* d'une branche du même dépôt ;
* d'un fork externe ;
* d'une source considérée non fiable.

### Pull request interne

Pour une pull request provenant du même dépôt :

* les secrets peuvent être disponibles selon les permissions ;
* les règles des environnements continuent de s'appliquer ;
* les secrets d'environnement ne sont accessibles qu'après validation des protections de l'environnement.

### Pull request externe

Pour une pull request provenant d'un fork ou d'une source non fiable :

* aucun secret de pipeline n'est injecté ;
* aucun secret de projet n'est injecté ;
* aucun secret d'environnement n'est injecté ;
* aucun job lié à un environnement protégé ne peut s'exécuter automatiquement.

Cette règle est contrôlée par Rustkins et ne peut pas être désactivée depuis le YAML.

---

## 9.4 Trigger `manual`

```yaml
on:
  manual:
    inputs:
      target:
        type: choice
        required: true
        options:
          - staging
          - production

      version:
        type: string
        required: true

      dry_run:
        type: boolean
        required: false
        default: false
```

Le trigger manuel est disponible depuis l'interface Rustkins.

### Types d'inputs supportés

```text
string
integer
boolean
choice
```

### Champs d'un input

| Champ | Obligatoire | Description |
|---|---:|---|
| `type` | Oui | Type de la valeur. |
| `required` | Non | Valeur obligatoire ou non. Défaut : `false`. |
| `default` | Non | Valeur par défaut. |
| `options` | Pour `choice` | Valeurs autorisées. |

### Contraintes

* le nom d'un input doit être unique ;
* `options` est obligatoire pour `choice` ;
* `default` doit être compatible avec le type ;
* un input obligatoire sans valeur ni valeur par défaut bloque le lancement.

---

## 9.5 Trigger `api`

```yaml
on:
  api:
    inputs:
      target:
        type: string
        required: true
```

Le trigger API utilise les mêmes règles d'inputs que le trigger manuel.

L'appel doit être authentifié avec un token disposant des permissions nécessaires.

Les permissions sont gérées par Rustkins, pas par le YAML.

---

## 10. Variables

Le champ `vars` contient des valeurs non sensibles.

### Variables globales

```yaml
vars:
  NODE_ENV: test
  RUST_LOG: info
```

### Variables par job

```yaml
jobs:
  test:
    vars:
      NODE_ENV: test
```

### Variables d'un service

```yaml
services:
  database:
    vars:
      POSTGRES_DB: application
```

### Ordre de surcharge

```text
variables globales du pipeline
→ variables du job
```

Les variables d'un service sont propres au conteneur de ce service.

### Contraintes

* objet clé/valeur ;
* clés non vides ;
* valeurs sous forme de chaînes après normalisation ;
* les informations sensibles doivent utiliser `secrets`.

---

## 11. Interpolation

La version 3 introduit une interpolation limitée.

Syntaxe :

```yaml
${{ namespace.value }}
```

Namespaces supportés :

```text
matrix
inputs
vars
```

Exemples :

```yaml
image: node:${{ matrix.node }}

vars:
  TARGET_ENV: ${{ inputs.target }}

steps:
  - run: echo "${{ vars.NODE_ENV }}"
```

### Emplacements autorisés

L'interpolation peut être utilisée notamment dans :

* `name` d'un job ;
* `image` d'un job ;
* valeurs de `vars` ;
* commandes `steps[].run` ;
* `cache.key.prefix` ;
* `cache.restore.prefixes` ;
* noms d'artifacts ;
* images et commandes de services.

### Restrictions

* aucune expression booléenne libre ;
* aucun appel de fonction ;
* aucune opération arithmétique ;
* aucun accès dynamique arbitraire ;
* l'interpolation n'est pas autorisée dans l'identifiant YAML d'un job ;
* l'interpolation n'est pas autorisée dans `needs`.

Une référence absente ou inconnue provoque une erreur de validation ou de compilation.

---

## 12. Secrets

La version 3 introduit les références de secrets.

Les valeurs des secrets ne sont jamais stockées dans le YAML.

Les secrets sont créés et gérés dans Rustkins.

### Portées supportées

```text
environnement
pipeline
projet
```

La portée est définie lors de la création du secret dans Rustkins.

Le YAML référence uniquement son nom.

---

## 12.1 Déclaration courte

```yaml
secrets:
  REGISTRY_TOKEN: registry-token
```

Cela signifie :

```text
variable injectée : REGISTRY_TOKEN
secret demandé    : registry-token
type d'injection  : variable d'environnement
required          : true
```

---

## 12.2 Déclaration longue

```yaml
secrets:
  OPTIONAL_TOKEN:
    from: optional-token
    required: false
```

Champs supportés :

| Champ | Obligatoire | Description |
|---|---:|---|
| `from` | Oui | Nom du secret dans Rustkins. |
| `required` | Non | Secret obligatoire. Défaut : `true`. |
| `target` | Non | Méthode d'injection. Défaut : variable d'environnement. |

---

## 12.3 Injection comme variable d'environnement

```yaml
secrets:
  REGISTRY_TOKEN:
    from: registry-token
    target:
      type: env
```

La variable injectée porte le nom de la clé YAML :

```text
REGISTRY_TOKEN=<secret>
```

La forme courte utilise automatiquement ce comportement.

---

## 12.4 Injection comme fichier

```yaml
secrets:
  KUBECONFIG:
    from: production-kubeconfig
    target:
      type: file
      path: /run/secrets/kubeconfig
```

### Règles

* le fichier est créé temporairement pour le job ;
* le chemin doit être absolu dans le conteneur ;
* le fichier n'est pas conservé dans le workspace ;
* le fichier est supprimé lors du nettoyage du job ;
* le contenu ne doit pas apparaître dans les logs ;
* les permissions du fichier doivent être restrictives.

---

## 12.5 Secrets globaux et par job

Un secret peut être demandé globalement :

```yaml
secrets:
  COMMON_TOKEN: common-token
```

Ou au niveau d'un job :

```yaml
jobs:
  publish:
    secrets:
      REGISTRY_TOKEN: registry-token
```

Les secrets du job sont fusionnés avec les secrets globaux.

En cas de conflit sur la même variable injectée, la déclaration du job surcharge la déclaration globale.

---

## 12.6 Secret obligatoire ou optionnel

Par défaut :

```yaml
required: true
```

Si un secret obligatoire n'est pas accessible :

```text
le job n'est pas dispatché
JobRun = failed
error_type = missing_secret
```

Pour un secret optionnel :

```yaml
secrets:
  OPTIONAL_TOKEN:
    from: optional-token
    required: false
```

Si le secret n'existe pas ou n'est pas accessible, aucune valeur n'est injectée.

---

## 12.7 Masquage dans les logs

Rustkins doit masquer les secrets dans :

* les logs persistés ;
* les chunks SSE ;
* les messages techniques ;
* les erreurs Docker ;
* les erreurs de steps ;
* les previews disponibles dans l'interface.

Le masquage doit être appliqué avant toute persistance ou diffusion.

---

## 13. Jobs

```yaml
jobs:
  build:
    image: node:22
    steps:
      - run: npm run build
```

### Contraintes

* au moins un job ;
* identifiants uniques ;
* identifiants stables ;
* un identifiant ne peut pas contenir d'interpolation ;
* chaque job doit avoir une image résolue ;
* chaque job doit contenir au moins une step.

### Nom affiché

La version 3 permet un nom lisible distinct de l'identifiant :

```yaml
jobs:
  test:
    name: Tests Node ${{ matrix.node }}
```

Le nom affiché peut utiliser l'interpolation matrix.

---

## 14. Dépendances entre jobs

```yaml
needs:
  - build
  - test
```

### Contraintes

* liste obligatoire si le champ est présent ;
* chaque job référencé doit exister ;
* cycles interdits ;
* interpolation interdite.

### Dépendance vers une matrix

Si `test` est un job matrix :

```yaml
package:
  needs:
    - test
```

Cela signifie que `package` dépend de toutes les variantes runtime de `test`.

Le job dépendant n'est évalué qu'après la fin de toutes les variantes.

Ce comportement constitue le fan-in automatique.

---

## 15. Politique d'exécution `run_if`

La version 3 conserve `run_if`.

```yaml
run_if: all_success
```

Valeurs supportées :

| Valeur | Description |
|---|---|
| `all_success` | Toutes les dépendances doivent être `success` ou `warning`. |
| `always` | Le job est évalué après la fin de toutes ses dépendances. |
| `on_failure` | Au moins une dépendance doit être `failed` ou `dead_lettered`. |

Valeur par défaut :

```yaml
run_if: all_success
```

`run_if` concerne uniquement les statuts des dépendances.

Les filtres Git et de contexte sont gérés par `when`.

---

## 16. Conditions avancées `when`

Le champ `when` contrôle l'exécution d'un job selon le contexte du run.

```yaml
when:
  branches:
    include:
      - main

  changed:
    include:
      - src/**
```

### Catégories supportées

```text
branches
tags
changed
trigger
```

### Combinaison

Les catégories présentes dans un même bloc `when` sont combinées avec un `AND`.

Exemple :

```yaml
when:
  branches:
    include:
      - main
  changed:
    include:
      - backend/**
```

Signification :

```text
branche compatible
ET
au moins un fichier backend modifié
```

À l'intérieur d'une liste, les valeurs sont combinées avec un `OR`.

---

## 16.1 Condition `branches`

```yaml
when:
  branches:
    include:
      - main
      - release/*
    exclude:
      - experimental/*
```

Le job est compatible si la branche correspond à au moins une inclusion et à aucune exclusion.

---

## 16.2 Condition `tags`

```yaml
when:
  tags:
    include:
      - v*
    exclude:
      - v*-alpha
```

Cette condition est compatible uniquement avec un run déclenché dans un contexte de tag.

---

## 16.3 Condition `trigger`

```yaml
when:
  trigger:
    - push
    - manual
```

Valeurs supportées :

```text
push
pull_request
manual
api
```

---

## 16.4 Condition `changed`

```yaml
when:
  changed:
    include:
      - src/**
      - package.json

    exclude:
      - docs/**
```

Les patterns sont relatifs à la racine du dépôt.

### Sémantique

* le job est sélectionné si au moins un fichier modifié correspond à `include` ;
* un fichier correspondant à `exclude` n'est pas pris en compte ;
* si aucun fichier restant ne correspond, le job est `skipped`.

### Diff sur un push

Rustkins compare :

```text
ancien SHA du push
→ nouveau SHA du push
```

### Diff sur une pull request

Rustkins compare :

```text
merge-base entre branche cible et branche source
→ SHA source de la pull request
```

### Diff impossible

Si Rustkins ne peut pas obtenir un diff fiable :

```text
job utilisant when.changed = skipped
```

Cette règle s'applique notamment si :

* l'ancien SHA est indisponible ;
* le merge-base est introuvable ;
* l'historique Git nécessaire est indisponible ;
* le provider ne fournit pas suffisamment d'informations.

Les autres jobs continuent normalement.

---

## 16.5 Évaluation combinée de `run_if` et `when`

Un job est exécutable uniquement si :

```text
run_if est satisfait
ET
when est satisfait
```

Si `when` n'est pas satisfait :

```text
JobRun = skipped
skip_reason = condition_not_matched
```

Si `when.changed` ne peut pas être évalué :

```text
JobRun = skipped
skip_reason = diff_unavailable
```

---

## 17. Matrix builds

La version 3 introduit les matrices statiques.

```yaml
strategy:
  matrix:
    node:
      - "20"
      - "22"

    database:
      - postgres
      - mariadb
```

Rustkins produit le produit cartésien :

```text
node=20, database=postgres
node=20, database=mariadb
node=22, database=postgres
node=22, database=mariadb
```

---

## 17.1 Identifiant logique et variantes runtime

L'identifiant YAML reste stable :

```yaml
jobs:
  test:
```

Les identifiants runtime sont générés par Rustkins :

```text
test[node=20,database=postgres]
test[node=20,database=mariadb]
test[node=22,database=postgres]
test[node=22,database=mariadb]
```

L'utilisateur ne peut pas interpoler l'identifiant YAML.

Le champ `name` peut être interpolé :

```yaml
name: Tests Node ${{ matrix.node }} / ${{ matrix.database }}
```

---

## 17.2 `exclude`

```yaml
strategy:
  matrix:
    node:
      - "20"
      - "22"

    database:
      - postgres
      - mariadb

    exclude:
      - node: "20"
        database: mariadb
```

Les variantes correspondant entièrement à une entrée `exclude` sont supprimées.

---

## 17.3 `include`

```yaml
strategy:
  matrix:
    node:
      - "20"
      - "22"

    database:
      - postgres

    include:
      - node: "23"
        database: postgres
        experimental: "true"
```

`include` permet :

* d'ajouter une combinaison absente du produit cartésien ;
* d'ajouter des propriétés supplémentaires à une variante.

---

## 17.4 Ordre d'expansion

Rustkins applique :

```text
1. validation des axes
2. génération du produit cartésien
3. application de exclude
4. application de include
5. détection des doublons
6. validation des limites
7. compilation des variantes runtime
```

---

## 17.5 Limites

```text
50 variantes maximum par job logique
200 jobs runtime maximum par pipeline
```

Les jobs non matrix comptent également dans la limite globale de 200 jobs runtime.

Une expansion dépassant une limite bloque la validation du pipeline.

---

## 17.6 Références matrix

```yaml
image: node:${{ matrix.node }}

vars:
  DATABASE_ENGINE: ${{ matrix.database }}
```

Une propriété référencée doit exister pour toutes les variantes dans lesquelles elle est utilisée.

Une référence absente provoque une erreur de compilation.

---

## 18. Ressources

La version 3 conserve les ressources CPU et mémoire :

```yaml
resources:
  cpu: 1000
  memory: 2048
```

### CPU

Exprimé en milli-CPU.

```text
500  = 0,5 CPU
1000 = 1 CPU
2000 = 2 CPU
```

### Mémoire

Exprimée en MiB.

```text
512  = 512 MiB
1024 = 1 GiB
2048 = 2 GiB
```

### Règles

* valeurs strictement positives ;
* limites appliquées au conteneur principal ;
* ressources utilisées par le scheduler ;
* les ressources des services doivent être prises en compte dans la capacité totale du job.

Une future extension pourra permettre des ressources spécifiques par service.

---

## 19. Timeout

```yaml
timeout: 600
```

Le timeout est exprimé en secondes et s'applique au job complet, y compris :

* démarrage des services ;
* attente des healthchecks ;
* restauration du cache ;
* téléchargement des artifacts ;
* exécution des steps ;
* upload des artifacts ;
* sauvegarde du cache.

Un timeout par step n'est pas supporté en version 3.

---

## 20. Steps

```yaml
steps:
  - name: Install dependencies
    run: npm ci

  - name: Run tests
    run: npm test
    continue_on_error: true
```

### Contraintes

* liste non vide ;
* exécution séquentielle ;
* `run` obligatoire ;
* `name` optionnel ;
* `continue_on_error` vaut `false` par défaut.

### Commandes multi-lignes

```yaml
steps:
  - name: Build and test
    run: |
      npm ci
      npm run build
      npm test
```

### `continue_on_error`

Si une step échoue avec :

```yaml
continue_on_error: true
```

Alors :

```text
StepRun = warning
le job continue
JobRun final = warning si aucune erreur bloquante ne survient
```

---

## 21. Jobs optionnels

```yaml
optional: true
```

Un job optionnel qui rencontre une erreur métier termine en `warning` au lieu de `failed`.

Une erreur d'infrastructure peut toujours déclencher les mécanismes de retry.

Valeur par défaut :

```yaml
optional: false
```

---

## 22. Services / sidecars

Un service est un conteneur auxiliaire lié à un job.

```yaml
services:
  database:
    image: postgres:16

    vars:
      POSTGRES_DB: application
      POSTGRES_USER: application

    secrets:
      POSTGRES_PASSWORD: postgres-test-password

    healthcheck:
      command: pg_isready -U application
      interval: 5
      timeout: 3
      retries: 10
```

Rustkins ne distingue pas les termes service et sidecar dans le modèle YAML.

---

## 22.1 Réseau isolé

Pour chaque job utilisant des services, Rustkins crée un réseau Docker temporaire dédié.

```text
réseau du job
├── conteneur principal
├── service database
└── autres services éventuels
```

Le service est accessible par son identifiant YAML :

```text
database:5432
```

Le réseau est supprimé à la fin du job.

Aucun port n'est publié sur l'hôte Docker.

---

## 22.2 Port personnalisé

Pour faire écouter un service sur un port différent, l'utilisateur configure directement le logiciel avec `command`.

Exemple MariaDB :

```yaml
services:
  mariadb-test:
    image: mariadb:11

    command:
      - --port=3307

    vars:
      MARIADB_USER: user
      MARIADB_DATABASE: mawster-testing

    secrets:
      MARIADB_PASSWORD: mariadb-user-password
      MARIADB_ROOT_PASSWORD: mariadb-root-password
```

Le conteneur principal utilise :

```text
mariadb-test:3307
```

Il n'existe pas de champ `ports` en version 3.

---

## 22.3 Commande d'un service

```yaml
command:
  - --port=3307
```

Le champ `command` remplace ou complète la commande de démarrage selon le comportement de l'image Docker.

Il est recommandé d'utiliser une liste de chaînes.

---

## 22.4 Variables et secrets d'un service

Les services supportent :

```yaml
vars:
  KEY: value

secrets:
  PASSWORD: secret-name
```

Les secrets du service ne sont injectés que dans ce service.

Ils ne sont pas automatiquement injectés dans le conteneur principal.

---

## 22.5 Healthcheck

Syntaxe simplifiée :

```yaml
healthcheck:
  command: pg_isready -U application
  interval: 5
  timeout: 3
  start_period: 10
  retries: 10
```

| Champ | Obligatoire | Description |
|---|---:|---|
| `command` | Oui | Commande de vérification. |
| `interval` | Non | Intervalle entre essais, en secondes. |
| `timeout` | Non | Timeout d'un essai, en secondes. |
| `start_period` | Non | Délai initial, en secondes. |
| `retries` | Non | Nombre maximal d'échecs. |

Rustkins attend que tous les services possédant un healthcheck deviennent sains avant d'exécuter la première step.

Si un service ne devient pas sain :

```text
JobRun = failed
error_type = service_unhealthy
```

---

## 22.6 Nettoyage des services

À la fin du job, Rustkins doit supprimer :

* les conteneurs de services ;
* le conteneur principal ;
* le réseau temporaire ;
* les volumes anonymes temporaires ;
* les fichiers de secrets temporaires.

Le nettoyage doit être tenté même après :

* échec ;
* annulation ;
* timeout ;
* crash récupérable.

---

## 22.7 Volumes

Les volumes hôte sont interdits.

Sont autorisés :

* volumes anonymes temporaires ;
* volumes gérés par Rustkins et limités à la durée du job.

Les volumes sont supprimés à la fin du job.

---

## 23. Cache

La version 3 introduit un cache déclaratif.

```yaml
cache:
  - name: npm

    scope: branch

    paths:
      - .npm/

    key:
      prefix: npm
      files:
        - package-lock.json

    restore:
      prefixes:
        - npm
```

Le cache est une optimisation.

Il ne doit pas être considéré comme un résultat officiel du pipeline.

---

## 23.1 Nom du cache

```yaml
name: npm
```

Le nom :

* identifie le cache dans les logs ;
* distingue plusieurs caches d'un job ;
* doit être unique dans le job.

---

## 23.2 Paths

```yaml
paths:
  - .npm/
  - node_modules/.cache/
```

Les chemins sont relatifs au workspace.

### Contraintes

* chemins relatifs uniquement ;
* chemins absolus interdits ;
* `../` interdit ;
* sortie du workspace interdite ;
* symlinks sortant du workspace interdits.

---

## 23.3 Construction de la clé

```yaml
key:
  prefix: npm
  files:
    - package-lock.json
```

Rustkins calcule un hash SHA-256 déterministe à partir :

* des chemins déclarés ;
* du contenu des fichiers ;
* de leur ordre normalisé.

Clé logique :

```text
<prefix>-<sha256>
```

Exemple :

```text
npm-4c9f8a76...
```

Une modification d'un fichier déclaré produit une nouvelle clé.

---

## 23.4 Scope

Valeurs supportées :

```text
branch
pipeline
project
```

### `branch`

Cache isolé par branche.

Valeur par défaut recommandée :

```yaml
scope: branch
```

### `pipeline`

Cache partagé entre les branches du même pipeline.

### `project`

Cache partagé entre les pipelines du même projet.

Un cache n'est jamais partagé entre plusieurs projets en version 3.

---

## 23.5 Restauration exacte

Avant les steps, Rustkins recherche d'abord la clé exacte.

Si elle existe :

* l'archive est téléchargée ;
* son intégrité est vérifiée ;
* les paths sont restaurés dans le workspace.

---

## 23.6 Restauration par préfixe

```yaml
restore:
  prefixes:
    - npm-node-${{ matrix.node }}
    - npm
```

Si la clé exacte n'existe pas, Rustkins essaie les préfixes dans l'ordre.

Pour chaque préfixe, il sélectionne le cache compatible créé le plus récemment.

Exemple :

```text
clé demandée : npm-B
cache exact absent
préfixe npm trouvé : npm-A
restauration de npm-A
```

Le job peut ensuite compléter le cache et créer la nouvelle clé `npm-B`.

---

## 23.7 Cache immuable

Une clé de cache existante ne peut jamais être remplacée.

Si plusieurs jobs tentent d'enregistrer la même clé :

```text
le premier upload valide crée le cache
les suivants ignorent l'upload
```

Cela évite les écrasements concurrents et les contenus variables pour une même clé.

---

## 23.8 Sauvegarde

Le cache est sauvegardé après un résultat :

```text
success
warning
```

Il n'est pas sauvegardé après :

```text
failed
cancelled
dead_lettered
```

Si la clé exacte existe déjà, aucun upload n'est effectué.

---

## 23.9 Erreur de cache

Une erreur de restauration ou de sauvegarde du cache ne fait pas échouer le job.

Elle produit un événement ou avertissement technique :

```text
cache_restore_failed
cache_save_failed
```

Le cache reste une optimisation non garantie.

---

## 23.10 Cache et matrix

```yaml
key:
  prefix: npm-node-${{ matrix.node }}
  files:
    - package-lock.json
```

Chaque variante peut ainsi disposer d'une clé distincte.

---

## 24. Artifacts

La version 3 remplace la forme unique de la version 2 par une liste d'artifacts nommés.

```yaml
artifacts:
  - name: application
    paths:
      - dist/**
    when: success
    retention: 30

  - name: test-reports
    paths:
      - reports/**
    when: always
    retention: 7
```

---

## 24.1 Champs d'un artifact

| Champ | Obligatoire | Description |
|---|---:|---|
| `name` | Oui | Nom logique de l'artifact. |
| `paths` | Oui | Fichiers et dossiers à archiver. |
| `when` | Non | Politique d'upload. Défaut : `success`. |
| `retention` | Non | Durée de rétention demandée, en jours. |

### Valeurs de `when`

```text
success
always
on_failure
```

`warning` est considéré comme compatible avec `success`.

### Rétention

La valeur demandée reste limitée par les politiques du projet et de l'instance.

Rustkins peut réduire une rétention demandée supérieure à la limite autorisée.

---

## 24.2 Archive

Chaque artifact est stocké sous forme d'archive `tar.gz`.

Rustkins conserve :

* nom ;
* job producteur ;
* variante matrix éventuelle ;
* taille ;
* checksum SHA-256 ;
* clé RustFS ;
* date d'expiration.

---

## 24.3 Artifacts et matrix

Chaque variante produit ses propres artifacts.

Exemples :

```text
test[node=20]/test-reports
test[node=22]/test-reports
```

Les artifacts de variantes différentes ne sont jamais fusionnés implicitement.

---

## 25. Téléchargement des artifacts

La consommation des artifacts est déclarée séparément :

```yaml
artifact_download:
  auto: false

  items:
    - from: build
      name: application
      path: ./release
```

---

## 25.1 Mode automatique

Pour conserver la compatibilité conceptuelle avec la version 2 :

```text
artifact_download absent
→ téléchargement automatique des artifacts des jobs listés dans needs
```

Forme explicite :

```yaml
artifact_download:
  auto: true
```

---

## 25.2 Désactivation du mode automatique

```yaml
artifact_download:
  auto: false
  items: []
```

Aucun artifact n'est restauré automatiquement.

---

## 25.3 Téléchargement sélectif

```yaml
artifact_download:
  auto: false

  items:
    - from: build
      name: application
      path: ./release
```

### Contraintes

* `from` doit référencer un job dans `needs` ;
* `name` doit correspondre à un artifact déclaré par le job source ;
* `path` doit rester dans le workspace ;
* les collisions de fichiers doivent être détectées.

---

## 25.4 Source matrix

Si `from` référence un job matrix, Rustkins télécharge les artifacts de toutes les variantes compatibles.

Ils sont restaurés dans des sous-dossiers séparés.

Exemple :

```text
./reports/test-node-20/
./reports/test-node-22/
```

Le nom du sous-dossier est généré de manière déterministe à partir de la variante runtime.

---

## 26. Environnements

Un job peut être lié à un environnement Rustkins :

```yaml
environment:
  name: production
  url: https://example.com
```

Un job lié à un environnement est considéré comme un job de déploiement.

Il reste exécuté comme un job Docker classique.

---

## 26.1 Environnement existant

L'environnement doit exister dans Rustkins avant l'exécution.

Le YAML ne peut pas créer automatiquement un environnement.

Si l'environnement est absent :

```text
JobRun = failed
error_type = environment_not_found
```

---

## 26.2 Protections

Les protections sont configurées côté Rustkins, par exemple :

* approbation obligatoire ;
* rôles autorisés ;
* branches autorisées ;
* secrets accessibles ;
* concurrence maximale.

Ces règles ne peuvent pas être affaiblies depuis le YAML.

---

## 26.3 Secrets d'environnement

Un job lié à un environnement peut demander des secrets de cet environnement.

Le secret doit toutefois être explicitement référencé dans `secrets`.

Rustkins n'injecte jamais automatiquement tous les secrets d'un environnement.

---

## 26.4 Concurrence

En version 3 :

```text
un seul job de déploiement actif par environnement
```

Les jobs suivants attendent dans une file FIFO.

Statut recommandé :

```text
waiting_environment
```

Le verrou est libéré lorsque le job atteint un état final :

```text
success
warning
failed
cancelled
dead_lettered
```

Le scheduler doit pouvoir récupérer et libérer un verrou abandonné après un crash.

---

## 26.5 URL

```yaml
environment:
  name: production
  url: https://example.com
```

L'URL est informative et peut être affichée dans l'interface et l'historique de déploiement.

---

## 27. Déploiements Swarm et Kubernetes

La version 3 n'introduit pas de DSL `deploy`.

Un déploiement est un job classique associé à un environnement.

Exemple Kubernetes :

```yaml
jobs:
  deploy:
    environment:
      name: production

    image: alpine/helm:latest

    secrets:
      KUBECONFIG:
        from: production-kubeconfig
        target:
          type: file
          path: /run/secrets/kubeconfig

    steps:
      - run: |
          helm upgrade --install application ./helm \
            --kubeconfig /run/secrets/kubeconfig
```

Exemple Docker Swarm :

```yaml
jobs:
  deploy:
    environment:
      name: production

    image: docker:cli

    secrets:
      DOCKER_CONFIG:
        from: production-docker-config
        target:
          type: file
          path: /run/secrets/docker-config.json

    steps:
      - run: docker stack deploy -c stack.yml application
```

Les credentials, protections et permissions sont gérés par Rustkins.

---

## 28. Gestion des erreurs

### Erreur métier

Exemples :

* commande retournant un code non nul ;
* tests échoués ;
* build invalide.

Comportement :

* échec normal du job ;
* pas de retry automatique ;
* `optional` et `continue_on_error` peuvent produire un warning.

### Erreur d'infrastructure

Exemples :

* Docker indisponible ;
* perte réseau ;
* échec temporaire RustFS ;
* worker perdu.

Comportement :

* retry selon la politique runtime ;
* passage possible en `dead_lettered`.

### Erreur de configuration

Exemples :

* secret obligatoire absent ;
* environnement inconnu ;
* matrix trop grande ;
* job inconnu dans `needs`.

Comportement :

* validation bloquée avant exécution ou job refusé avant dispatch selon le moment de détection.

---

## 29. Statuts supplémentaires V3

### PipelineRun

```text
queued
running
waiting_approval
success
warning
failed
cancelled
```

### JobRun

```text
waiting_dependencies
waiting_environment
waiting_approval
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

### Service runtime

```text
creating
starting
healthy
unhealthy
stopped
removed
```

### Cache operation

```text
miss
restored_exact
restored_prefix
saved
already_exists
failed
```

---

## 30. Normalisation

La normalisation transforme le YAML en modèle interne complet.

Exemple :

```yaml
jobs:
  test:
    image: node:22

    cache:
      - name: npm
        paths:
          - .npm/
        key:
          prefix: npm
          files:
            - package-lock.json

    steps:
      - run: npm test
```

Après normalisation :

```text
job.id = "test"
job.name = "test"
job.image = "node:22"
job.needs = []
job.run_if = all_success
job.optional = false
job.timeout = default_timeout
job.resources = default_resources
job.vars = merge(global_vars, job_vars)
job.secrets = merge(global_secrets, job_secrets)
job.when = always_match
job.strategy = no_matrix
job.services = []
job.cache[0].scope = branch
job.cache[0].restore.prefixes = []
job.artifact_download.auto = true
job.environment = none
```

---

## 31. Compilation matrix

Après normalisation, un job matrix est expansé en plusieurs définitions runtime.

Exemple :

```yaml
jobs:
  test:
    strategy:
      matrix:
        node: ["20", "22"]
```

Compilation :

```text
LogicalJobDefinition("test")
  → RuntimeJobDefinition("test[node=20]")
  → RuntimeJobDefinition("test[node=22]")
```

Chaque variante reçoit :

* ses valeurs matrix ;
* son image interpolée ;
* ses variables interpolées ;
* ses caches résolus ;
* ses services résolus ;
* ses artifacts résolus.

---

## 32. Modèle interne indicatif

```rust
PipelineDefinition {
    version: 3,
    name: "Application CI/CD",
    default_image: Some("node:22"),
    triggers: vec![
        Trigger::Push { /* ... */ },
        Trigger::PullRequest {
            checkout: PullRequestCheckout::Merge,
            /* ... */
        },
        Trigger::Manual { /* inputs */ },
        Trigger::Api { /* inputs */ },
    ],
    vars: HashMap::new(),
    secrets: vec![],
    jobs: vec![
        LogicalJobDefinition {
            id: "test",
            display_name: Some(
                "Tests Node ${{ matrix.node }}"
            ),
            needs: vec![],
            run_if: RunPolicy::AllSuccess,
            when: WhenCondition::default(),
            optional: false,
            image: "node:${{ matrix.node }}",
            strategy: Some(MatrixStrategy {
                axes: /* ... */,
                include: /* ... */,
                exclude: /* ... */,
            }),
            services: vec![
                ServiceDefinition {
                    id: "database",
                    image: "postgres:16",
                    command: vec![],
                    vars: HashMap::new(),
                    secrets: vec![],
                    healthcheck: Some(HealthcheckDefinition {
                        command: "pg_isready -U application",
                        interval_seconds: 5,
                        timeout_seconds: 3,
                        start_period_seconds: 0,
                        retries: 10,
                    }),
                }
            ],
            caches: vec![
                CacheDefinition {
                    name: "npm",
                    scope: CacheScope::Branch,
                    paths: vec![".npm/"],
                    key_prefix: "npm",
                    key_files: vec!["package-lock.json"],
                    restore_prefixes: vec!["npm"],
                }
            ],
            artifacts: vec![],
            artifact_download: ArtifactDownloadPolicy::Auto,
            environment: None,
            steps: vec![],
        }
    ],
}
```

---

## 33. Schéma logique

```yaml
version: 3

name: string

image: string

on:
  push:
    branches:
      include:
        - string
      exclude:
        - string
    tags:
      include:
        - string
      exclude:
        - string

  pull_request:
    branches:
      include:
        - string
      exclude:
        - string
    actions:
      - opened | synchronize | reopened
    checkout: head | merge

  manual:
    inputs:
      <input_name>:
        type: string | integer | boolean | choice
        required: boolean
        default: any
        options:
          - any

  api:
    inputs:
      <input_name>:
        type: string | integer | boolean | choice
        required: boolean
        default: any
        options:
          - any

vars:
  KEY: string

secrets:
  ENV_NAME: secret-name

jobs:
  <job_id>:
    name: string

    needs:
      - <job_id>

    run_if: all_success | always | on_failure

    when:
      branches:
        include:
          - string
        exclude:
          - string

      tags:
        include:
          - string
        exclude:
          - string

      changed:
        include:
          - string
        exclude:
          - string

      trigger:
        - push | pull_request | manual | api

    strategy:
      matrix:
        <axis>:
          - string

        include:
          - object

        exclude:
          - object

    image: string

    optional: boolean

    vars:
      KEY: string

    secrets:
      ENV_NAME: secret-name

    resources:
      cpu: integer
      memory: integer

    timeout: integer

    services:
      <service_id>:
        image: string
        command:
          - string
        vars:
          KEY: string
        secrets:
          ENV_NAME: secret-name
        healthcheck:
          command: string
          interval: integer
          timeout: integer
          start_period: integer
          retries: integer

    cache:
      - name: string
        scope: branch | pipeline | project
        paths:
          - string
        key:
          prefix: string
          files:
            - string
        restore:
          prefixes:
            - string

    steps:
      - name: string
        run: string
        continue_on_error: boolean

    artifacts:
      - name: string
        paths:
          - string
        when: success | always | on_failure
        retention: integer

    artifact_download:
      auto: boolean
      items:
        - from: <job_id>
          name: string
          path: string

    environment:
      name: string
      url: string
```

---

## 34. Contraintes non exprimées par un schéma JSON simple

Les validations suivantes doivent être réalisées par le moteur Rustkins :

* les jobs référencés dans `needs` doivent exister ;
* le DAG ne doit contenir aucun cycle ;
* chaque job doit avoir une image résolue ;
* les identifiants YAML des jobs ne peuvent pas contenir d'interpolation ;
* les références matrix doivent exister dans toutes les variantes concernées ;
* l'expansion matrix ne doit pas dépasser 50 variantes par job ;
* le pipeline ne doit pas dépasser 200 jobs runtime ;
* une dépendance vers un job matrix représente toutes ses variantes ;
* les secrets obligatoires doivent être accessibles avant le dispatch ;
* les secrets d'une pull request externe ne doivent jamais être injectés ;
* les environnements doivent exister ;
* les protections d'environnement doivent être appliquées hors du YAML ;
* un seul job de déploiement peut détenir le verrou d'un environnement ;
* les chemins de cache et d'artifacts doivent rester dans le workspace ;
* les services ne peuvent pas publier de port sur l'hôte ;
* les volumes hôte sont interdits ;
* les caches sont immuables ;
* une clé de cache existante ne doit jamais être écrasée ;
* les artifacts de matrix doivent être séparés par variante ;
* les téléchargements sélectifs doivent référencer des jobs dépendants ;
* `when.changed` non évaluable doit produire `skipped` ;
* `run_if` et `when` doivent être tous les deux satisfaits ;
* les secrets doivent être masqués avant persistance ou diffusion des logs.

---

## 35. Fonctionnalités hors périmètre version 3

Les fonctionnalités suivantes ne sont pas supportées :

```text
expressions booléennes libres
langage d'expression complet
fonctions comme hash_files()
needs dynamiques
identifiants de job dynamiques
matrix dynamique générée par un job
matrix dépassant les limites configurées
volumes hôte dans les services
publication de ports de services sur l'hôte
services persistants entre plusieurs jobs
cache partagé entre plusieurs projets
cache mutable
DSL de déploiement Kubernetes natif
DSL de déploiement Docker Swarm natif
création d'environnements depuis le YAML
désactivation des protections d'environnement depuis le YAML
secrets injectés dans les pull requests externes
timeout par step
variables par step
```

---

## 36. Validation du pipeline

Avant exécution, Rustkins réalise :

1. lecture du YAML ;
2. validation syntaxique ;
3. validation de la version ;
4. validation des champs ;
5. validation des types ;
6. validation des enums ;
7. validation des triggers ;
8. validation des inputs ;
9. validation des références de jobs ;
10. détection des cycles ;
11. validation des conditions `when` ;
12. validation des matrices ;
13. contrôle des limites d'expansion ;
14. validation des services ;
15. validation des caches ;
16. validation des artifacts ;
17. validation des environnements référencés ;
18. normalisation ;
19. expansion matrix ;
20. interpolation ;
21. compilation vers le modèle interne ;
22. création d'une `PipelineVersion` immuable.

---

## 37. Critères de validation de la spécification v3

La spécification v3 est considérée correctement implémentée si :

1. Un pipeline `version: 3` est parsé et validé.
2. Les pipelines versions 1 et 2 restent supportés.
3. Les secrets courts et longs sont acceptés.
4. Les secrets globaux et de job sont fusionnés.
5. Un secret peut être injecté comme variable.
6. Un secret peut être injecté comme fichier.
7. Un secret obligatoire absent bloque le job.
8. Un secret optionnel absent ne bloque pas le job.
9. Les secrets sont masqués avant les logs persistés et SSE.
10. Les secrets externes ne sont pas injectés dans une PR de fork.
11. `when` combine les catégories avec `AND`.
12. Les valeurs d'une liste `when` sont combinées avec `OR`.
13. `when.changed` fonctionne sur un push.
14. `when.changed` fonctionne sur une pull request.
15. Un diff impossible produit un job `skipped`.
16. `run_if` et `when` sont évalués ensemble.
17. Une matrix produit le produit cartésien attendu.
18. `exclude` retire les variantes correspondantes.
19. `include` ajoute les variantes demandées.
20. Une matrix dépassant 50 variantes est refusée.
21. Un pipeline dépassant 200 jobs runtime est refusé.
22. Une dépendance vers un job matrix attend toutes ses variantes.
23. Le nom affiché d'un job peut utiliser `matrix`.
24. L'identifiant YAML d'un job reste statique.
25. Un réseau isolé est créé pour chaque job avec services.
26. Les services sont accessibles par DNS interne.
27. Aucun port de service n'est publié sur l'hôte.
28. Un service peut changer son port via `command`.
29. Rustkins attend les healthchecks avant les steps.
30. Un service unhealthy fait échouer le job.
31. Les conteneurs, réseaux et volumes temporaires sont nettoyés.
32. Un cache exact est restauré.
33. Un cache de fallback est restauré par préfixe.
34. Le scope branch, pipeline et project est respecté.
35. Une clé de cache existante n'est jamais écrasée.
36. Un cache est sauvegardé après success ou warning.
37. Une erreur de cache ne fait pas échouer le job.
38. Les artifacts nommés sont uploadés.
39. La politique `artifacts[].when` est respectée.
40. La rétention demandée est limitée par la politique Rustkins.
41. `artifact_download.auto` conserve le comportement automatique.
42. Un artifact peut être téléchargé sélectivement.
43. Les artifacts matrix sont restaurés dans des sous-dossiers distincts.
44. Un environnement doit exister avant le run.
45. Un job lié à un environnement est enregistré comme déploiement.
46. Un seul déploiement actif est autorisé par environnement.
47. Les déploiements suivants attendent en FIFO.
48. Les protections d'environnement sont appliquées côté Rustkins.
49. Les triggers sur tags fonctionnent.
50. Les triggers pull request fonctionnent.
51. Le checkout `head` fonctionne.
52. Le checkout `merge` fonctionne.
53. Un conflit de merge termine le run avec `merge_conflict`.
54. Les inputs manuels sont validés.
55. Les inputs API sont validés.
56. Les valeurs `matrix`, `inputs` et `vars` sont interpolées.
57. Les expressions libres sont rejetées.
58. Les jobs de déploiement Kubernetes et Swarm peuvent être réalisés avec des CLI ordinaires.

---

## 38. Évolutions futures

Les fonctionnalités suivantes pourront être étudiées dans une version ultérieure :

* langage d'expression typé ;
* opérateurs booléens ;
* fonctions d'interpolation ;
* matrix dynamique ;
* réutilisation de jobs ;
* templates de pipelines ;
* composants de pipeline ;
* services persistants partagés ;
* cache distribué inter-projets contrôlé ;
* politiques avancées de cache ;
* DSL de déploiement ;
* rollback intégré ;
* approbations déclaratives ;
* priorités et quotas ;
* affinités de workers ;
* images prébuildées Rustkins ;
* registry interne.

---

## 39. Résumé

La version 3 transforme la spécification Rustkins en format CI/CD avancé tout en conservant un modèle déclaratif contrôlable.

Principes structurants :

```text
- compatibilité avec les versions précédentes
- secrets référencés, jamais stockés dans Git
- interpolation volontairement limitée
- conditions when sans expressions libres
- matrix statique avec limites strictes
- fan-in automatique
- services temporaires sur réseau isolé
- aucun port publié sur l'hôte
- cache immuable et non bloquant
- artifacts nommés et téléchargements sélectifs
- environnements créés et protégés côté Rustkins
- déploiements exécutés comme jobs classiques
- pull requests sécurisées
- PostgreSQL reste la source de vérité runtime
```
