# Rustkins Pipeline Specification v2 (V1 Scope)

## 1. Introduction

Ce document décrit la version 2 de la spécification YAML des pipelines Rustkins.

Cette version correspond au périmètre fonctionnel de la V1 du produit Rustkins.

La version 2 étend la version 1 avec des fonctionnalités nécessaires à un usage en équipe :

* variables globales ;
* variables par job ;
* déclenchement API ;
* ressources CPU/RAM par job ;
* artifacts ;
* politique d'exécution avec `run_if`.

Les fonctionnalités lourdes comme les secrets, le cache, les services/sidecars, la matrix et les conditions avancées `when` restent hors périmètre de cette version.

---

## 2. Compatibilité avec la version 1

Un pipeline utilisant cette spécification doit déclarer :

```yaml
version: 2
```

La version 2 conserve les concepts de la version 1 :

* `name`
* `on`
* `jobs`
* `image`
* `needs`
* `timeout`
* `steps`
* `steps[].name`
* `steps[].run`

Les pipelines en `version: 1` restent valides, mais ne peuvent pas utiliser les nouveaux champs de la version 2.

---

## 3. Emplacement du fichier

Les pipelines Rustkins sont stockés dans le dossier :

```text
.rustkins/
```

Chaque pipeline doit respecter le format :

```text
.rustkins/<nom>.pipeline.yml
```

Exemples :

```text
.rustkins/backend.pipeline.yml
.rustkins/frontend.pipeline.yml
.rustkins/release.pipeline.yml
```

---

## 4. Exemple complet

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
  RUST_LOG: info

jobs:
  build:
    image: node:22

    resources:
      cpu: 1000
      memory: 2048

    timeout: 600

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

    image: node:22

    vars:
      NODE_ENV: test

    resources:
      cpu: 1000
      memory: 1024

    timeout: 600

    steps:
      - name: Run tests
        run: npm test

  cleanup:
    needs:
      - build
      - test

    run_if: always

    image: alpine:latest

    steps:
      - name: Cleanup
        run: echo "cleanup"
```

---

## 5. Structure globale

Un fichier pipeline version 2 doit respecter la structure suivante :

```yaml
version: integer
name: string
on: object
vars: object
jobs: object
```

## 5.1 Champs racine

| Champ     | Type    | Obligatoire | Description                           |
| --------- | ------- | ----------- | ------------------------------------- |
| `version` | integer | Oui         | Version de la spécification utilisée. |
| `name`    | string  | Oui         | Nom lisible du pipeline.              |
| `on`      | object  | Oui         | Déclencheurs du pipeline.             |
| `vars`    | object  | Non         | Variables globales du pipeline.       |
| `jobs`    | object  | Oui         | Liste des jobs du pipeline.           |

---

## 6. Version

Le champ `version` indique la version de la spécification utilisée.

```yaml
version: 2
```

## Contraintes

* obligatoire ;
* doit être un entier ;
* doit valoir `2` pour utiliser les fonctionnalités de cette spécification.

## Erreurs possibles

```text
Pipeline version is missing.
Unsupported pipeline version: 3.
Field "resources" requires pipeline version 2.
```

---

## 7. Nom du pipeline

Le champ `name` représente le nom affiché dans l'interface utilisateur.

```yaml
name: Backend CI
```

## Contraintes

* obligatoire ;
* chaîne non vide ;
* longueur maximale recommandée : 100 caractères.

---

## 8. Déclencheurs

Le champ `on` définit les événements pouvant démarrer le pipeline.

La version 2 supporte :

* `push`
* `manual`
* `api`

---

## 8.1 Trigger `push`

Le trigger `push` permet de lancer un pipeline lorsqu'un commit est poussé sur une branche autorisée.

```yaml
on:
  push:
    branches:
      - main
      - develop
```

## Contraintes

* `branches` est obligatoire ;
* `branches` doit être une liste ;
* au moins une branche doit être définie ;
* chaque branche doit être une chaîne non vide.

---

## 8.2 Trigger `manual`

Le trigger `manual` permet de lancer un pipeline manuellement depuis l'interface Rustkins.

```yaml
on:
  manual: {}
```

## Contraintes

* doit être un objet vide ;
* ne prend aucun paramètre en version 2.

---

## 8.3 Trigger `api`

Le trigger `api` permet de lancer un pipeline depuis un système externe via l'API Rustkins.

```yaml
on:
  api: {}
```

Exemple d'usage :

```text
POST /projects/{project_id}/pipelines/{pipeline_id}/runs
```

## Contraintes

* doit être un objet vide ;
* l'appel API doit être authentifié par un token valide ;
* la gestion des permissions est réalisée côté API Rustkins, pas dans le YAML.

---

## 8.4 Combinaison de triggers

Plusieurs triggers peuvent être définis dans un même pipeline.

```yaml
on:
  push:
    branches:
      - main

  manual: {}

  api: {}
```

Un pipeline doit contenir au moins un trigger.

---

## 9. Variables

La version 2 introduit le champ `vars`.

Les variables permettent de définir des valeurs non sensibles disponibles pendant l'exécution des jobs.

Exemple :

```yaml
vars:
  NODE_ENV: test
  RUST_LOG: info
```

## Contraintes

* optionnel ;
* doit être un objet clé/valeur ;
* les clés doivent être des chaînes non vides ;
* les valeurs doivent être des chaînes ;
* les secrets ne doivent pas être stockés dans `vars`.

---

## 9.1 Variables globales

Les variables globales sont définies à la racine du pipeline.

```yaml
vars:
  NODE_ENV: test
  API_URL: https://api.example.local
```

Elles sont injectées dans tous les jobs du pipeline.

---

## 9.2 Variables par job

Un job peut définir ses propres variables.

```yaml
jobs:
  test:
    image: node:22

    vars:
      NODE_ENV: test

    steps:
      - run: npm test
```

Les variables du job sont fusionnées avec les variables globales.

En cas de conflit, la variable du job surcharge la variable globale.

Exemple :

```yaml
vars:
  NODE_ENV: production

jobs:
  test:
    image: node:22

    vars:
      NODE_ENV: test
```

Résultat pour le job `test` :

```text
NODE_ENV=test
```

---

## 9.3 Exclusion des secrets

Les variables ne sont pas des secrets.

Il est interdit d'utiliser `vars` pour stocker :

* mots de passe ;
* tokens ;
* clés API ;
* clés privées ;
* credentials Docker ;
* credentials cloud.

Les secrets seront traités dans une version ultérieure de la spécification.

---

## 10. Jobs

Le champ `jobs` contient l'ensemble des jobs du pipeline.

```yaml
jobs:
  build:
    image: node:22
    steps:
      - run: npm ci
```

## Contraintes

* obligatoire ;
* doit contenir au moins un job ;
* chaque identifiant de job doit être unique ;
* un identifiant de job doit être stable, car il peut être référencé par `needs`.

## Recommandation de nommage

Utiliser uniquement :

```text
a-z
0-9
-
_
```

Exemples valides :

```text
build
test
build-api
unit_tests
```

---

## 11. Dépendances entre jobs

Le champ `needs` permet de définir les dépendances d'un job.

```yaml
jobs:
  test:
    needs:
      - build

    image: node:22

    steps:
      - run: npm test
```

## Contraintes

* optionnel ;
* doit être une liste ;
* chaque élément doit référencer un job existant ;
* les cycles sont interdits.

Exemple invalide :

```yaml
needs: build
```

`needs` doit toujours être une liste, même avec une seule dépendance.

---

## 12. Environnement d'exécution

Chaque job est exécuté dans un conteneur Docker.

Le champ `image` définit l'image utilisée.

Une image peut être définie globalement au niveau du pipeline. Dans ce cas, elle est utilisée par défaut pour tous les jobs qui ne définissent pas explicitement leur propre image.

```yaml
image: node:22

jobs:
  build:
    steps:
      - run: npm ci

  test:
    image: node:20

    steps:
      - run: npm test
```

Dans cet exemple :

* le job `build` utilise l'image globale `node:22` ;
* le job `test` utilise sa propre image `node:20`.

## Contraintes

* une image doit être définie soit globalement, soit au niveau de chaque job ;
* si une image globale est définie, elle peut être surchargée par un job ;
* les valeurs doivent être des chaînes non vides ;
* une seule image est utilisée par job après résolution.

---

## 13. Ressources

La version 2 introduit le champ `resources`.

Ce champ permet de définir les ressources réservées pour un job.

```yaml
jobs:
  build:
    image: node:22

    resources:
      cpu: 1000
      memory: 2048
```

## 13.1 CPU

Le champ `cpu` est exprimé en milli-CPU.

```yaml
resources:
  cpu: 1000
```

Exemples :

| Valeur | Signification |
| ------ | ------------- |
| `500`  | 0,5 CPU       |
| `1000` | 1 CPU         |
| `2000` | 2 CPU         |

## Contraintes

* optionnel ;
* entier strictement positif ;
* si absent, Rustkins applique une valeur par défaut.

---

## 13.2 Mémoire

Le champ `memory` est exprimé en MiB.

```yaml
resources:
  memory: 2048
```

Exemples :

| Valeur | Signification |
| ------ | ------------- |
| `512`  | 512 MiB       |
| `1024` | 1 GiB         |
| `2048` | 2 GiB         |

## Contraintes

* optionnel ;
* entier strictement positif ;
* si absent, Rustkins applique une valeur par défaut.

---

## 13.3 Utilisation par le scheduler

Les ressources déclarées servent à deux choses :

1. limiter le conteneur Docker au moment de son exécution ;
2. permettre au scheduler Rustkins de répartir les jobs entre les workers.

Exemple :

```yaml
jobs:
  build:
    image: rust:latest

    resources:
      cpu: 2000
      memory: 4096

    steps:
      - run: cargo build --release
```

Le scheduler doit vérifier qu'un worker dispose des ressources suffisantes avant d'y envoyer le job.

---

## 14. Timeout

Chaque job peut définir un timeout.

```yaml
timeout: 600
```

La valeur est exprimée en secondes.

## Contraintes

* optionnel ;
* entier strictement positif ;
* le timeout s'applique au job complet ;
* aucun timeout par step n'est supporté en version 2.

Si aucun timeout n'est fourni, Rustkins applique un timeout par défaut.

---

## 15. Steps

Chaque job doit contenir une liste ordonnée d'étapes.

```yaml
steps:
  - run: npm ci
  - run: npm test
```

## Contraintes

* obligatoire ;
* au moins une étape doit être présente ;
* les steps sont exécutés séquentiellement ;
* si une step échoue, le job échoue ;
* les steps suivantes du même job ne sont pas exécutées.

---

## 15.1 Champ `run`

Le champ `run` définit la commande exécutée.

```yaml
steps:
  - run: npm ci
```

## Contraintes

* obligatoire ;
* chaîne non vide.

---

## 15.2 Champ `name`

Le champ `name` permet de donner un nom lisible à l'étape.

```yaml
steps:
  - name: Install dependencies
    run: npm ci
```

## Contraintes

* optionnel ;
* chaîne non vide si présent.

---

## 15.3 Commandes multi-lignes

Les commandes multi-lignes sont autorisées.

```yaml
steps:
  - name: Build and test
    run: |
      npm ci
      npm run build
      npm test
```

Le moteur exécute le contenu de `run` comme un script shell dans le conteneur du job.

---

## 16. Artifacts

La version 2 introduit les artifacts.

Un artifact est un fichier ou dossier produit par un job et sauvegardé par Rustkins après l'exécution du job.

Exemple :

```yaml
jobs:
  build:
    image: node:22

    steps:
      - run: npm ci
      - run: npm run build

    artifacts:
      paths:
        - dist/**
```

## 16.1 Upload d'artifacts

Le champ `artifacts.paths` définit les fichiers ou dossiers à sauvegarder.

```yaml
artifacts:
  paths:
    - dist/**
    - coverage/**
```

## Contraintes

* `artifacts` est optionnel ;
* `paths` est obligatoire si `artifacts` est présent ;
* `paths` doit être une liste ;
* chaque path doit être une chaîne non vide ;
* les patterns glob simples sont autorisés.

---

## 16.2 Téléchargement automatique des artifacts

En version 2, un job télécharge automatiquement les artifacts des jobs listés dans `needs`.

Exemple :

```yaml
jobs:
  build:
    image: rust:latest

    steps:
      - run: cargo build --release

    artifacts:
      paths:
        - target/release/my-app

  package:
    needs:
      - build

    image: alpine:latest

    steps:
      - run: tar -czf my-app.tar.gz target/release/my-app
```

Dans cet exemple :

1. le job `build` produit `target/release/my-app` ;
2. Rustkins sauvegarde ce fichier comme artifact ;
3. le job `package` dépend de `build` ;
4. Rustkins restaure automatiquement les artifacts de `build` avant d'exécuter `package`.

---

## 16.3 Limites des artifacts en version 2

La version 2 ne définit pas encore :

* de nom explicite d'artifact ;
* de durée de rétention par pipeline ;
* de téléchargement sélectif ;
* de partage d'artifacts entre pipelines différents.

Ces fonctionnalités pourront être ajoutées dans une version ultérieure.

---

## 17. Politique d'exécution `run_if`

La version 2 introduit le champ `run_if`.

Il permet de contrôler l'exécution d'un job selon le statut de ses dépendances.

```yaml
jobs:
  cleanup:
    needs:
      - test

    run_if: always

    image: alpine:latest

    steps:
      - run: echo "cleanup"
```

## 17.1 Valeurs supportées

| Valeur        | Description                                                                        |
| ------------- | ---------------------------------------------------------------------------------- |
| `all_success` | Le job s'exécute uniquement si toutes ses dépendances ont réussi.                  |
| `always`      | Le job s'exécute dès que ses dépendances sont terminées, même si elles ont échoué. |
| `on_failure`  | Le job s'exécute uniquement si au moins une dépendance a échoué.                   |

## 17.2 Valeur par défaut

Si `run_if` est absent, Rustkins applique :

```yaml
run_if: all_success
```

---

## 17.3 Exemple `always`

```yaml
jobs:
  test:
    image: node:22

    steps:
      - run: npm test

  cleanup:
    needs:
      - test

    run_if: always

    image: alpine:latest

    steps:
      - run: echo "cleanup"
```

Dans cet exemple, `cleanup` est exécuté même si `test` échoue.

---

## 17.4 Exemple `on_failure`

```yaml
jobs:
  test:
    image: node:22

    steps:
      - run: npm test

  failure-report:
    needs:
      - test

    run_if: on_failure

    image: alpine:latest

    steps:
      - run: echo "Tests failed"
```

Dans cet exemple, `failure-report` est exécuté uniquement si `test` échoue.

---

## 17.5 Cas d'un job sans dépendance

Pour un job sans `needs`, `run_if` n'a pas d'effet pratique.

Un job sans dépendance est prêt à être exécuté dès le début du pipeline.

---

## 18. Gestion des erreurs d'exécution

Si un job échoue :

* son statut devient `failed` ;
* les jobs dépendants avec `run_if: all_success` passent `skipped` ;
* les jobs dépendants avec `run_if: always` peuvent s'exécuter ;
* les jobs dépendants avec `run_if: on_failure` peuvent s'exécuter.

Exemple :

```text
build -> failed

test:
  needs: [build]
  run_if: all_success
  result: skipped

cleanup:
  needs: [build]
  run_if: always
  result: success

failure-report:
  needs: [build]
  run_if: on_failure
  result: success
```

---

## 19. Fonctionnalités hors périmètre version 2

Les fonctionnalités suivantes ne sont pas supportées dans cette version :

```text
secrets
cache
services
sidecars
matrix
when
conditions par branche au niveau job
conditions par fichiers modifiés
timeout par step
image globale
variables par step
artifacts nommés
artifacts entre pipelines
pull request trigger
tag trigger
```

L'utilisation de ces champs doit provoquer une erreur de validation.

---

## 20. Validation du pipeline

Avant exécution, Rustkins effectue les validations suivantes :

1. lecture du YAML ;
2. validation de la structure ;
3. validation des champs obligatoires ;
4. validation des types ;
5. validation des enums ;
6. validation des dépendances ;
7. détection des cycles ;
8. normalisation des valeurs par défaut ;
9. compilation vers le modèle interne.

---

## 21. Normalisation

La normalisation transforme le YAML utilisateur en modèle complet prêt à être exécuté.

Exemple YAML :

```yaml
jobs:
  test:
    needs:
      - build

    image: node:22

    steps:
      - run: npm test
```

Après normalisation :

```text
job.id = "test"
job.image = "node:22"
job.needs = ["build"]
job.timeout = default_timeout
job.run_if = all_success
job.resources.cpu = default_cpu
job.resources.memory = default_memory
job.vars = merge(global_vars, job_vars)
```

---

## 22. Modèle interne

Après validation, le YAML est transformé en représentation interne.

Exemple :

```rust
PipelineDefinition {
    version: 2,
    name: "Backend CI",
    triggers: vec![
        Trigger::Push {
            branches: vec!["main", "develop"]
        },
        Trigger::Manual,
        Trigger::Api
    ],
    vars: HashMap::from([
        ("NODE_ENV", "test"),
        ("RUST_LOG", "info")
    ]),
    jobs: vec![
        JobDefinition {
            id: "build",
            image: "node:22",
            needs: vec![],
            run_if: RunPolicy::AllSuccess,
            timeout_seconds: Some(600),
            resources: ResourceRequirements {
                cpu_millis: Some(1000),
                memory_mib: Some(2048)
            },
            vars: HashMap::new(),
            steps: vec![
                StepDefinition {
                    name: Some("Install dependencies"),
                    run: "npm ci"
                }
            ],
            artifacts: Some(ArtifactDefinition {
                paths: vec!["dist/**"]
            })
        }
    ]
}
```

---

## 23. Schéma logique

```yaml
version: 2

name: string

on:
  push:
    branches:
      - string

  manual: {}

  api: {}

vars:
  KEY: string

jobs:
  <job_id>:
    needs:
      - <job_id>

    run_if: all_success | always | on_failure

    image: string

    vars:
      KEY: string

    resources:
      cpu: integer
      memory: integer

    timeout: integer

    steps:
      - name: string
        run: string

    artifacts:
      paths:
        - string
```

---

## 24. Schéma JSON simplifié

```json
{
  "type": "object",
  "required": ["version", "name", "on", "jobs"],
  "properties": {
    "version": {
      "type": "integer",
      "const": 2
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
                "type": "string",
                "minLength": 1
              }
            }
          },
          "additionalProperties": false
        },
        "manual": {
          "type": "object",
          "maxProperties": 0
        },
        "api": {
          "type": "object",
          "maxProperties": 0
        }
      },
      "additionalProperties": false
    },
    "vars": {
      "type": "object",
      "additionalProperties": {
        "type": "string"
      }
    },
    "jobs": {
      "type": "object",
      "minProperties": 1,
      "additionalProperties": {
        "type": "object",
        "required": ["image", "steps"],
        "properties": {
          "needs": {
            "type": "array",
            "items": {
              "type": "string",
              "minLength": 1
            }
          },
          "run_if": {
            "type": "string",
            "enum": ["all_success", "always", "on_failure"]
          },
          "image": {
            "type": "string",
            "minLength": 1
          },
          "vars": {
            "type": "object",
            "additionalProperties": {
              "type": "string"
            }
          },
          "resources": {
            "type": "object",
            "properties": {
              "cpu": {
                "type": "integer",
                "minimum": 1
              },
              "memory": {
                "type": "integer",
                "minimum": 1
              }
            },
            "additionalProperties": false
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
              "required": ["run"],
              "properties": {
                "name": {
                  "type": "string",
                  "minLength": 1
                },
                "run": {
                  "type": "string",
                  "minLength": 1
                }
              },
              "additionalProperties": false
            }
          },
          "artifacts": {
            "type": "object",
            "required": ["paths"],
            "properties": {
              "paths": {
                "type": "array",
                "minItems": 1,
                "items": {
                  "type": "string",
                  "minLength": 1
                }
              }
            },
            "additionalProperties": false
          }
        },
        "additionalProperties": false
      }
    }
  },
  "additionalProperties": false
}
```

---

## 25. Contraintes non exprimées par le schéma JSON

Certaines règles doivent être validées par le moteur Rustkins :

* les jobs référencés dans `needs` doivent exister ;
* le graphe des dépendances ne doit contenir aucun cycle ;
* les identifiants de jobs doivent être uniques ;
* les artifacts des jobs dans `needs` doivent être restaurés avant l'exécution du job dépendant ;
* les variables globales et les variables de job doivent être fusionnées correctement ;
* les ressources demandées doivent pouvoir être admises par au moins un worker ;
* un job sans `needs` ne doit pas dépendre du résultat d'un autre job pour son `run_if`.

---

## 26. Évolution future

Les fonctionnalités suivantes sont candidates pour une version 3 de la spécification :

* `when` ;
* cache ;
* services/sidecars ;
* secrets ;
* matrix ;
* triggers pull request ;
* triggers tags ;
* artifacts nommés ;
* téléchargement sélectif des artifacts ;
* variables par step.
