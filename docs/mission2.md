---
icon: lucide/globe
---

# Mission 2 : Application Web Laravel — GSBRH

## Présentation

### Objectif

**GSBRH** est une application web développée avec le framework **Laravel** (PHP). Elle sert de système central de gestion des ressources humaines pour le réseau GSB (Groupement de Santé Bretagne).

Elle propose deux interfaces complémentaires :
- Une **interface web** (Blade + Bootstrap) réservée aux membres RH pour gérer les praticiens
- Une **API REST sécurisée par JWT** consommée par l'application mobile Flutter (Mission 3)

Les fonctionnalités principales sont :
- Authentification sécurisée (session web + token JWT pour l'API)
- Consultation et recherche des praticiens
- Gestion de l'ancienneté et des échelons salariaux
- Système de notation des praticiens (notes clients et experts)
- Gestion des demandes de congé
- Notifications internes entre RH et praticiens

---

## Stack Technique

| Élément | Technologie |
|--------|-------------|
| Langage | PHP 8.2+ |
| Framework | Laravel 12.0 |
| Base de données | MySQL (production) / SQLite (développement) |
| Authentification API | JWT — `tymon/jwt-auth ^2.2` |
| Frontend | Blade Templates + Bootstrap 5.3.2 |
| Bundler | Vite |
| Documentation API | L5 Swagger (OpenAPI 3.0) |
| Tests | PHPUnit 11.5.3 |
| ORM | Eloquent (intégré Laravel) |

---

## Architecture

```
GSBRH/
├── app/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── AuthController.php          # API : login, register, logout, refresh, me
│   │   │   ├── PostController.php          # API : CRUD praticiens + notes
│   │   │   ├── PraticienControlleur.php    # Web : liste, recherche, ancienneté
│   │   │   └── ConnexionControlleur.php    # Web : connexion / déconnexion
│   │   └── Resources/
│   │       └── PraticienResource.php       # Sérialisation JSON des praticiens
│   └── Models/
│       ├── Praticien.php
│       ├── Connexion.php                   # Utilisateur JWT
│       ├── Note.php
│       ├── Congé.php
│       ├── Notification.php
│       ├── Expert.php
│       ├── Echelon.php
│       ├── Ville.php
│       ├── TypePraticien.php
│       ├── Etat.php
│       └── EtatLecture.php
├── routes/
│   ├── api.php                             # Routes REST (JWT)
│   └── web.php                             # Routes web (session)
├── resources/views/
│   ├── praticiens/
│   │   ├── index.blade.php                 # Liste des praticiens
│   │   └── show.blade.php                  # Détail praticien
│   └── Connecter/
│       ├── index.blade.php                 # Formulaire de connexion
│       └── fraude.blade.php                # Page d'avertissement fraude
└── database/migrations/                    # Schéma de la base de données
```

---

## Base de données

### Entités principales

#### Praticien

| Champ | Type | Description |
|-------|------|-------------|
| `id` | int (PK) | Identifiant unique |
| `nom` | string(50) | Nom de famille |
| `prenom` | string(60) | Prénom |
| `adresse` | string(100) | Adresse postale |
| `coef_notoriete` | float | Coefficient de notoriété |
| `code_type_praticien` | string(6) (FK) | Type de praticien |
| `id_ville` | int (FK) | Ville rattachée |
| `Solde_congé` | decimal | Solde de congé actuel |
| `Ancien_Solde_Congé` | decimal | Solde de l'année précédente |
| `anciennete` | int | Années d'ancienneté |
| `id_echelon` | int (FK) | Échelon salarial |
| `note_client` | float | Note moyenne clients |
| `note_expert` | float | Note moyenne experts |
| `note_global` | float | Note globale calculée |

#### Connexion (utilisateurs)

| Champ | Type | Description |
|-------|------|-------------|
| `identifiant` | string(50) (PK) | Email de connexion |
| `mdp` | string | Mot de passe haché (BCrypt) |
| `id_praticiens` | int (FK) | Praticien associé |
| `privilèges` | int | 1 = Admin RH, 2 = Praticien standard |

#### Congé

| Champ | Type | Description |
|-------|------|-------------|
| `ID` | int (PK) | Identifiant |
| `Id_praticien` | int (FK) | Praticien demandeur |
| `date_debut` | datetime | Date de début |
| `date_fin` | datetime | Date de fin |
| `état` | int (FK) | 1=En attente, 2=Accepté, 3=Refusé |

#### Note

| Champ | Type | Description |
|-------|------|-------------|
| `id` | int (PK) | Identifiant |
| `commentaires` | string | Texte du commentaire |
| `note` | int (1-10) | Score attribué |
| `id_praticiens` | int (FK, nullable) | Client auteur — `null` si note d'expert |
| `id_expert` | int (FK, nullable) | Expert auteur — `null` si note client |
| `concerne` | int (FK) | Praticien noté |

#### Echelon

| Champ | Type | Description |
|-------|------|-------------|
| `id_echelon` | int (PK) | Identifiant de l'échelon |
| `duree` | int | Durée dans cet échelon (en années) |
| `salaire_brut` | float | Salaire brut correspondant |

#### Notification

| Champ | Type | Description |
|-------|------|-------------|
| `id_notif` | int (PK) | Identifiant |
| `id_ecrivian` | int | Expéditeur |
| `id_receveur` | int (FK) | Praticien destinataire |
| `message` | string | Contenu du message |
| `id_etat` | int (FK) | 1=Lu, 2=Non lu |

---

## Interface Web

### Connexion

La page de connexion (`/login`) permet aux membres RH de s'authentifier via leur **adresse mail** et **mot de passe**.

Le système vérifie les identifiants avec `Hash::check()` (BCrypt), puis ouvre une session Laravel. En cas de tentative suspecte répétée, une page d'alerte fraude (`fraude.blade.php`) est affichée.

### Liste des praticiens

La page `/praticiens` affiche tous les praticiens sous forme de grille Bootstrap. Chaque carte affiche le nom, le prénom et la ville du praticien.

Un champ de **recherche par nom** (insensible à la casse) est disponible via un formulaire `POST /search`.

### Détail d'un praticien

La page `/praticiens/{id}` affiche la fiche complète d'un praticien :
- Informations personnelles (nom, prénom, adresse, ville)
- Ancienneté actuelle
- Échelon et salaire brut associé
- Notes clients et experts

Un bouton **"Ajouter 1 an d'ancienneté"** permet d'incrémenter l'ancienneté directement depuis l'interface (`POST /praticiens/{id}/add-anciennete`).

---

## API REST

L'API REST expose des endpoints consommés par l'application mobile Flutter.  
La documentation complète est générée automatiquement et accessible à : `/api/documentation`

### Authentification

| Méthode | Endpoint | Accès | Description |
|---------|----------|-------|-------------|
| POST | `/api/auth/login` | Public | Connexion → retourne un token JWT |
| POST | `/api/auth/register` | Public | Création d'un compte |
| POST | `/api/auth/logout` | JWT | Déconnexion (révocation du token) |
| POST | `/api/auth/refresh` | JWT | Renouvellement du token |
| GET | `/api/auth/me` | JWT | Profil de l'utilisateur connecté |

#### Exemple de réponse login (200 OK)

```json
{
  "status": "success",
  "user": {
    "identifiant": "jean.dupont@gsb.fr",
    "id_praticiens": 1,
    "privilèges": 1
  },
  "authorization": {
    "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "type": "bearer",
    "expires_in": 3600
  }
}
```

#### Exemple d'erreur (401 Unauthorized)

```json
{
  "status": "error",
  "message": "Identifiants incorrects"
}
```

### Praticiens

| Méthode | Endpoint | Accès | Description |
|---------|----------|-------|-------------|
| GET | `/api/praticiens` | Public | Liste complète des praticiens |
| GET | `/api/praticiens/{id}` | Public | Détail d'un praticien |
| GET | `/api/praticiens/nom/{nom}` | Public | Recherche par nom |
| POST | `/api/praticiens` | JWT | Créer un praticien |
| PUT | `/api/praticiens/{id}` | JWT | Modifier un praticien |
| DELETE | `/api/praticiens/{id}` | JWT | Supprimer un praticien |

**Champs requis à la création / modification :**  
`nom`, `prenom`, `adresse`, `id_ville`, `anciennete`, `id_echelon`  
Les clés étrangères (`id_ville`, `id_echelon`) sont validées par une vérification d'existence en base.

#### Exemple de réponse praticien

```json
{
  "data": {
    "id": 1,
    "nom": "Dupont",
    "prenom": "Jean",
    "adresse": "123 Rue de Paris",
    "ville": {
      "id": 35001,
      "nom_reel": "Rennes",
      "code_postal": "35000"
    },
    "note_client": 4.5,
    "note_expert": 4.8,
    "note_global": 4.65
  }
}
```

### Notes

| Méthode | Endpoint | Accès | Description |
|---------|----------|-------|-------------|
| GET | `/api/notes` | Public | Toutes les notes |
| GET | `/api/notes/{id}` | Public | Notes d'un praticien spécifique |
| POST | `/api/note/create` | JWT | Soumettre une nouvelle note |

---

## Logique Métier Clé

### Ancienneté

L'ancienneté d'un praticien est incrémentée manuellement par le RH via le bouton dédié dans l'interface web. Ce champ détermine l'échelon salarial correspondant.

```php title='Incrémenter l'ancienneté'
$praticien = Praticien::findOrFail($id);
$praticien->anciennete += 1;
$praticien->save();
```

### Recherche de praticien

La recherche s'effectue avec une requête SQL `LIKE` insensible à la casse :

```php title='Recherche par nom'
$praticiens = Praticien::where('nom', 'LIKE', '%' . $request->search . '%')->get();
```

### Authentification JWT (API)

L'API utilise `tymon/jwt-auth`. Après connexion, le token est requis dans le header `Authorization` pour toutes les routes protégées :

```
Authorization: Bearer <token>
```

Les routes protégées utilisent le middleware `auth:api`. Les routes publiques (consultation des praticiens et des notes) ne nécessitent pas de token.

Le payload JWT embarque des **claims personnalisés** pour identifier l'utilisateur côté mobile :

```php title='Connexion.php — Claims JWT'
public function getJWTCustomClaims(): array
{
    return [
        'id_praticiens' => $this->id_praticiens,
        'privileges'    => $this->privilèges,
    ];
}
```
