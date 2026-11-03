---
id: DOC-0017
title: "Rustkins Pipeline Specification v1 (V0 Scope)"
type: specification
status: draft
summary: "Contrat du format de pipeline YAML v1 de Rustkins V0 : déclencheurs, jobs, dépendances, image d'exécution et steps."
topics: [pipeline, yaml, jobs, triggers]
applies_to: [V0]
specification_version: "1"
---

# Rustkins Pipeline Specification v1 (V0 Scope)

## 1. Introduction

Ce document décrit la spécification officielle du format de pipeline utilisé par Rustkins.

Le fichier pipeline constitue l'interface publique du moteur CI. Il définit :

* les déclencheurs du pipeline ;
* les jobs à exécuter ;
* les dépendances entre les jobs ;
* l'environnement d'exécution des jobs ;
* les étapes exécutées dans chaque job.

Cette spécification correspond au périmètre fonctionnel de la V0 de Rustkins.

---

## 2. Emplacement du fichier

Les définitions de pipelines doivent être stockées dans le répertoire :

```text
.rustkins/
```

Chaque pipeline est défini dans un fichier YAML situé sous ce
répertoire.

Le nom du fichier est libre. La seule contrainte de chemin est que le
fichier soit placé sous `.rustkins/`.

Exemples :

```text
.rustkins/ci.yml
.rustkins/backend.yml
.rustkins/frontend.yml
```

Plusieurs pipelines peuvent coexister dans un même dépôt.

---

## 3. Exemple complet

```yaml
version: 1

name: Backend CI

on:
  push:
    branches:
      - main

  manual: {}

jobs:

  build:

    image: node:22

    timeout: 600

    steps:

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

  test:

    needs:
      - build

    image: node:22

    steps:

      - name: Run tests
        run: npm test
```

---

## 4. Structure globale

Un fichier pipeline doit respecter la structure suivante :

```yaml
version: integer
name: string
on: object
jobs: object
```

## 4.1 Champs racine

| Champ   | Type    | Obligatoire |
| ------- | ------- | ----------- |
| version | integer | Oui         |
| name    | string  | Oui         |
| on      | object  | Oui         |
| jobs    | object  | Oui         |

---

## 5. Version

Le champ `version` indique la version de la spécification utilisée.

### Exemple

```yaml
version: 1
```

### Contraintes

* obligatoire ;
* doit être un entier ;
* seule la valeur `1` est supportée en V0.

### Erreurs possibles

```text
Pipeline version is missing.
Unsupported pipeline version: 2.
```

---

## 6. Nom du pipeline

Le champ `name` représente le nom affiché dans l'interface utilisateur.

### Exemple

```yaml
name: Backend CI
```

### Contraintes

* obligatoire ;
* chaîne non vide ;
* longueur maximale recommandée : 100 caractères.

### Erreurs possibles

```text
Pipeline name is missing.
Pipeline name cannot be empty.
```

---

## 7. Déclencheurs

Le champ `on` définit les événements pouvant démarrer le pipeline.

La V0 supporte uniquement :

* `push`
* `manual`

## 7.1 Déclenchement sur push

### Exemple

```yaml
on:
  push:
    branches:
      - main
      - develop
```

### Contraintes

* `branches` est obligatoire ;
* au moins une branche doit être définie.

### Exemple valide

```yaml
on:
  push:
    branches:
      - main
```

---

## 7.2 Déclenchement manuel

### Exemple

```yaml
on:
  manual: {}
```

Le pipeline pourra être lancé depuis l'interface utilisateur ou l'API Rustkins.

---

## 7.3 Combinaison de triggers

Plusieurs déclencheurs peuvent être définis.

Exemple :

```yaml
on:
  push:
    branches:
      - main

  manual: {}
```

---

## 8. Jobs

Le champ `jobs` contient l'ensemble des jobs du pipeline.

Chaque job possède un identifiant unique.

### Exemple

```yaml
jobs:
  build:
    ...

  test:
    ...
```

## Contraintes

* au moins un job doit être défini ;
* chaque identifiant doit être unique.

## Recommandations

Utiliser uniquement :

```text
a-z
0-9
-
_
```

Exemples :

```text
build
build-api
unit_tests
```

Exemples déconseillés :

```text
"Mon super job"
"Build API Production"
```

---

## 9. Dépendances entre jobs

Les dépendances sont définies à l'aide du champ `needs`.

### Exemple

```yaml
jobs:

  test:

    needs:
      - build
```

## Contraintes

* optionnel ;
* doit toujours être une liste ;
* chaque job référencé doit exister.

Exemple valide :

```yaml
needs:
  - build
  - lint
```

Exemple invalide :

```yaml
needs: build
```

---

## 9.1 Cycles interdits

Les dépendances cycliques sont interdites.

Exemple invalide :

```yaml
jobs:

  build:
    needs:
      - test

  test:
    needs:
      - build
```

Erreur :

```text
Circular dependency detected between jobs.
```

---

## 10. Environnement d'exécution

Chaque job est exécuté dans un conteneur Docker.

Le champ `image` définit l'image utilisée.

### Exemple

```yaml
jobs:

  build:
    image: node:22
```

## Contraintes

* obligatoire ;
* chaîne non vide.

Erreur :

```text
Job "build" is missing image.
```

---

## 11. Timeout

Chaque job peut définir un timeout.

### Exemple

```yaml
timeout: 600
```

La valeur est exprimée en secondes.

## Contraintes

* optionnel ;
* entier strictement positif.

Si aucun timeout n'est fourni, Rustkins applique un timeout par défaut.

Erreur :

```text
Invalid timeout value.
```

---

## 12. Steps

Chaque job doit contenir une liste ordonnée d'étapes.

### Exemple

```yaml
steps:

  - run: npm ci

  - run: npm test
```

## Contraintes

* obligatoire ;
* au moins une étape doit être présente.

---

## 12.1 Champ `run`

Définit la commande exécutée.

### Exemple

```yaml
- run: npm ci
```

Le champ est obligatoire.

Erreur :

```text
Step is missing run command.
```

---

## 12.2 Champ `name`

Permet de donner un nom lisible à l'étape.

### Exemple

```yaml
- name: Install dependencies
  run: npm ci
```

Le champ est optionnel.

---

## 12.3 Commandes multi-lignes

Les scripts multi-lignes sont autorisés.

Exemple :

```yaml
steps:

  - name: Build and test
    run: |
      npm ci
      npm run build
      npm test
```

---

## 13. Gestion des erreurs d'exécution

Si un job échoue :

* son statut devient `failed` ;
* les jobs dépendants ne sont pas exécutés ;
* les jobs dépendants passent au statut `skipped`.

Exemple :

```text
build -> failed

test -> skipped
deploy -> skipped
```

---

## 14. Fonctionnalités hors périmètre V0

Les fonctionnalités suivantes ne sont pas supportées :

```text
resources
artifacts
cache
services
secrets
variables d'environnement
matrix
conditions d'exécution avancées
run_if
trigger API
trigger pull request
trigger tags
```

L'utilisation de ces champs doit provoquer une erreur de validation.

---

## 15. Validation du pipeline

Avant exécution, Rustkins effectue les validations suivantes :

1. lecture du YAML ;
2. validation du schéma ;
3. validation des dépendances ;
4. détection des cycles ;
5. génération du modèle interne.

Si une erreur est détectée, l'exécution du pipeline est refusée.

---

## 16. Modèle interne

Après validation, le fichier YAML est transformé en représentation interne.

Exemple :

```rust
PipelineDefinition {
    version: 1,
    name: "Backend CI",
    jobs: vec![
        JobDefinition {
            id: "build",
            image: "node:22",
            timeout: Some(600),
            needs: vec![],
            steps: vec![
                StepDefinition {
                    name: Some("Install dependencies"),
                    run: "npm ci"
                }
            ]
        }
    ]
}
```

---

## 17. Schéma logique de la spécification

Le schéma ci-dessous décrit la structure attendue d'un pipeline Rustkins V1 (périmètre V0).

### Structure générale

```yaml
version: integer

name: string

on:
  push:
    branches:
      - string
  manual: {}

jobs:
  <job_id>:
    needs:
      - <job_id>

    image: string

    timeout: integer

    steps:
      - name: string
        run: string
```

### Schéma JSON simplifié

```json
{
  "type": "object",
  "required": [
    "version",
    "name",
    "on",
    "jobs"
  ],
  "properties": {
    "version": {
      "type": "integer",
      "const": 1
    },

    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },

    "on": {
      "type": "object",
      "minProperties": 1,
      "properties": {
        "push": {
          "type": "object",
          "required": ["branches"],
          "properties": {
            "branches": {
              "type": "array",
              "minItems": 1,
              "items": {
                "type": "string"
              }
            }
          },
          "additionalProperties": false
        },

        "manual": {
          "type": "object"
        }
      },
      "additionalProperties": false
    },

    "jobs": {
      "type": "object",
      "minProperties": 1,
      "additionalProperties": {
        "type": "object",
        "required": [
          "image",
          "steps"
        ],

        "properties": {
          "needs": {
            "type": "array",
            "items": {
              "type": "string"
            }
          },

          "image": {
            "type": "string",
            "minLength": 1
          },

          "timeout": {
            "type": "integer",
            "minimum": 1
          },

          "steps": {
            "type": "array",
            "minItems": 1,

            "items": {
              "type": "object",

              "required": [
                "run"
              ],

              "properties": {
                "name": {
                  "type": "string"
                },

                "run": {
                  "type": "string",
                  "minLength": 1
                }
              },

              "additionalProperties": false
            }
          }
        },

        "additionalProperties": false
      }
    }
  },

  "additionalProperties": false
}
```

### Contraintes supplémentaires non exprimables dans le schéma

Certaines règles doivent être validées par le moteur Rustkins après la validation du schéma :

* les identifiants référencés dans `needs` doivent correspondre à des jobs existants ;
* le graphe des dépendances ne doit contenir aucun cycle ;
* au moins un trigger doit être défini dans `on` ;
* les identifiants de jobs doivent être uniques ;
* les jobs dépendants d'un job en échec doivent être marqués `skipped`.

Ces validations sont réalisées lors de la phase de compilation du pipeline.
