---
id: DOC-0016
title: "Spécifications Rustkins"
type: specification
status: approved
summary: "Index des contrats et spécifications du produit, notamment les trois versions du format YAML des pipelines."
topics: [specifications, pipeline, yaml]
applies_to: [all]
---

# Spécifications

Décrivent **le comportement attendu** : règles fonctionnelles, contrats, formats et validations. Ne pas y stocker des arbitrages historiques ou un journal d'erreurs.

## Formats de pipeline

- [YAML v1 (périmètre Rustkins V0)](pipeline/yaml-v1.md)
- [YAML v2 (périmètre Rustkins V1)](pipeline/yaml-v2.md)
- [YAML v3 (périmètre Rustkins V2)](pipeline/yaml-v3.md)

Les spécifications API et fonctionnelles seront ajoutées dans ce dossier quand des contrats autonomes seront rédigés. Un contenu transversal ne doit pas être dupliqué dans chaque version : déclarer son périmètre de compatibilité.
