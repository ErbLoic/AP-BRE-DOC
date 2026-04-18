---
icon: lucide/rocket
---

# Accueil

## Introduction

Ceci est la documentation officielle des missions **AP SIO 2** pour la région Bretagne — année scolaire **2025-2026**.

Réalisée par **Djerba Ywel** et **Erbuer Loïc**.

Ce projet comprend trois applications interconnectées autour d'un système de gestion RH pour le réseau **GSB (Groupement de Santé Bretagne)**. Ces applications partagent la même base de données MySQL et le même domaine métier : la gestion des praticiens de santé.

---

## Infrastructure réseau (Mission 0)

Lors de cette mission nous avons mis en place une infrastructure simple de réseau. Voici un tableau récapitulatif des rôles et adresses IP :

| Rôle | Adresse IPv4 |
|------|--------------|
| Serveur Web (Laravel) | 172.23.48.1 |
| Serveur BDD (MySQL) | 172.23.48.2 |
| Poste développeur 1 | 172.23.48.10 |
| Poste développeur 2 | 172.23.48.11 |
| Poste Production | 172.23.48.20 |

---

## Vue d'ensemble des missions

### Mission 1 — GSBConge (Application Desktop C#)

Application **Windows Forms (.NET 8)** permettant la gestion des demandes de congé :
- Les praticiens soumettent des demandes de congé avec dates de début/fin
- Le RH consulte les demandes en attente et les **accepte ou refuse**
- Les praticiens reçoivent des **notifications** du RH à leur prochaine connexion
- Le solde de congé est mis à jour automatiquement via des **procédures stockées MySQL**

[Voir la documentation complète →](mission1.md)

---

### Mission 2 — GSBRH (Application Web Laravel)

Application web **Laravel 12 / PHP 8.2** servant de système RH central :
- Interface web Bootstrap pour les membres RH (consultation, recherche, ancienneté)
- **API REST avec JWT** consommée par l'application mobile Flutter
- Gestion des praticiens : informations, échelons salariaux, ancienneté
- Système de notation (notes clients et experts)
- Documentation API auto-générée via **Swagger / OpenAPI**

[Voir la documentation complète →](mission2.md)

---

### Mission 3 — GSBNote (Application Mobile Flutter)

Application **Flutter / Dart** multi-plateforme (Android, iOS, Windows…) :
- Consultation de la liste des praticiens avec **recherche en temps réel**
- Affichage du détail d'un praticien avec ses notes **clients** et **experts**
- Soumission de nouvelles notes (commentaire + score de 1 à 10)
- Authentification **JWT** via l'API GSBRH

[Voir la documentation complète →](mission3.md)

---

## Schéma d'intégration

```
┌─────────────────────┐        ┌──────────────────────────┐
│   GSBConge (C#)     │        │     GSBNote (Flutter)    │
│   Desktop Windows   │        │  Mobile / Multi-platform │
└────────┬────────────┘        └────────────┬─────────────┘
         │ SQL direct                        │ HTTP REST + JWT
         │                                   │
         ▼                                   ▼
┌─────────────────────────────────────────────────────────┐
│              MySQL — 172.23.48.2 — BDD gsb              │
│                                                         │
│  praticien · congé · connexion · note · notification    │
│  echelon · ville · expert · etat · etat_lecture         │
└────────────────────────────┬────────────────────────────┘
                             │ Eloquent ORM
                             ▼
                  ┌─────────────────────┐
                  │    GSBRH (Laravel)  │
                  │    172.23.48.1      │
                  │  API REST + Web UI  │
                  └─────────────────────┘
```
