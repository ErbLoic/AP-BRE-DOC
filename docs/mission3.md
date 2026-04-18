---
icon: lucide/smartphone
---

# Mission 3 : Application Mobile Flutter — GSBNote

## Présentation

### Objectif

**GSBNote** est une application mobile multi-plateforme développée avec **Flutter / Dart**. Elle permet de consulter les praticiens du réseau GSB (Groupement de Santé Bretagne) ainsi que leurs notes et commentaires, émis soit par des **clients**, soit par des **experts**.

Les fonctionnalités principales sont :
- Connexion sécurisée avec **JWT** (token fourni par l'API GSBRH)
- Consultation de la liste des praticiens avec recherche en temps réel
- Affichage du détail d'un praticien (profil + notes)
- Séparation des notes **clients** et **experts** via des onglets
- Soumission d'une nouvelle note (commentaire + score) — réservée aux utilisateurs connectés
- Gestion des erreurs réseau avec écrans dédiés

> GSBNote ne possède **aucune base de données locale**. Toute la donnée est fournie par l'**API REST de GSBRH** hébergée sur `http://172.23.48.1`.

---

## Stack Technique

| Élément | Technologie |
|--------|-------------|
| Langage | Dart |
| Framework | Flutter 3.9.2+ |
| HTTP Client | `http ^0.13.6` |
| Authentification | JWT Bearer Token |
| UI | Material Design 3 |
| Plateformes supportées | Android, iOS, Windows, macOS, Linux |
| Gestion d'état | `StatefulWidget` (réactif) |

---

## Architecture

```
GSBNote/
├── lib/
│   ├── main.dart             # Point d'entrée (widget Appreciations)
│   ├── auth_service.dart     # Service d'authentification JWT (singleton)
│   ├── login_page.dart       # Écran de connexion
│   ├── home_screen.dart      # Liste des praticiens (Accueil)
│   └── details.dart          # Détail praticien + onglets notes
├── pubspec.yaml              # Dépendances et configuration Flutter
├── android/                  # Config Android
├── ios/                      # Config iOS
├── windows/                  # Config Windows
├── macos/                    # Config macOS
├── linux/                    # Config Linux
└── test/                     # Tests unitaires
```

### Hiérarchie des écrans

Au démarrage, l'application charge directement l'écran d'accueil. Si l'utilisateur tente une action nécessitant une authentification (ex. soumettre une note), il est redirigé vers la page de connexion. Une fois authentifié, il peut accéder aux fonctionnalités complètes.

```
main() → Appreciations → Accueil (liste praticiens)
                              │
                              ├── [non connecté] → LoginPage → retour Accueil
                              │
                              └── [tap praticien] → Details (fiche + notes)
                                                        └── [connecté] → Formulaire ajout note
```

---

## Modèles de données (réponses API)

### Praticien

```json
{
  "id": 1,
  "nom": "Dupont",
  "prenom": "Jean",
  "adresse": "123 Rue de Paris",
  "ville": {
    "id": 35001,
    "nom_ville": "RENNES",
    "nom_reel": "Rennes",
    "code_postal": "35000"
  },
  "note_client": 4.5,
  "note_expert": 4.8,
  "note_global": 4.65
}
```

### Note

```json
{
  "id": 12,
  "commentaires": "Très bon praticien, à l'écoute.",
  "note": 9,
  "id_praticiens": 3,
  "id_expert": null,
  "concerne": 1
}
```

> Si `id_expert` est `null` → c'est une note **client**.  
> Si `id_praticiens` est `null` → c'est une note **expert**.

---

## Endpoints API consommés

| Méthode | Endpoint | Auth | Description |
|---------|----------|------|-------------|
| POST | `/api/auth/login` | Public | Connexion → retourne le JWT |
| GET | `/api/praticiens` | Public | Liste tous les praticiens |
| GET | `/api/praticiens/{id}` | Public | Détail d'un praticien |
| GET | `/api/praticiens/search?search={query}` | Public | Recherche par nom |
| GET | `/api/notes/{id}` | Public | Notes d'un praticien |
| POST | `/api/note/create` | JWT | Soumettre une nouvelle note |

---

## Description des écrans

### Écran de connexion (`login_page.dart`)

L'écran de connexion affiche deux champs : **email** et **mot de passe**. Lors de la soumission, une requête `POST /api/auth/login` est envoyée à l'API GSBRH. En cas de succès, le **token JWT** est stocké dans le singleton `AuthService` et l'utilisateur est renvoyé vers l'écran précédent. En cas d'échec (identifiants incorrects ou erreur réseau), un message d'erreur est affiché via une `SnackBar`.

### Accueil — Liste des praticiens (`home_screen.dart`)

L'écran principal affiche la liste complète des praticiens sous forme de **cartes Material**. Chaque carte affiche :
- Nom et prénom du praticien
- Note globale (avec icône étoile)
- Ville de rattachement

Un champ de recherche permet de filtrer les résultats **en temps réel** via l'endpoint `GET /api/praticiens/search?search=`. Si aucun résultat n'est trouvé, un écran "aucun résultat" est affiché. En cas d'erreur réseau, un écran d'erreur dédié est présenté avec un bouton pour réessayer.

Un `PopupMenuButton` dans l'AppBar permet à l'utilisateur de se déconnecter ou de consulter son profil.

### Détail d'un praticien (`details.dart`)

La page de détail affiche la fiche complète du praticien sélectionné. Elle comprend :
- Un **SliverAppBar** avec fond en dégradé (couleur primaire → secondaire) qui se replie au défilement
- Les informations personnelles du praticien (adresse, ville)
- Ses trois scores : note client, note expert, note globale
- Un **TabBar** avec deux onglets :
  - **Notes Clients** : commentaires laissés par des clients (filtrés par `id_expert == null`)
  - **Notes Experts** : évaluations laissées par des experts (filtrés par `id_expert != null`)

Un **bouton flottant (FAB)** permet d'ouvrir un formulaire modal pour soumettre une nouvelle note (champ commentaire + score de 1 à 10). Cette action nécessite d'être connecté — si ce n'est pas le cas, l'utilisateur est redirigé vers la page de connexion.

---

## Logique Métier Clé

### Service d'authentification (`auth_service.dart`)

`AuthService` est un **singleton** qui centralise la gestion du token JWT sur toute la durée de vie de l'application :

```dart title='auth_service.dart — Structure'
class AuthService {
  static final AuthService _instance = AuthService._internal();
  factory AuthService() => _instance;
  AuthService._internal();

  String? _token;
  String? _userEmail;
  int? _userId;
  int? _privileges;

  bool get isLoggedIn => _token != null;

  Future<bool> login(String email, String password) async {
    // POST /api/auth/login
    // Stocke le token en cas de succès
  }

  Map<String, String> getAuthHeaders() {
    return {
      'Authorization': 'Bearer $_token',
      'Content-Type': 'application/json',
    };
  }

  void logout() {
    _token = null;
    _userEmail = null;
    _userId = null;
  }
}
```

### Filtrage des notes (client vs expert)

Lors du chargement des notes d'un praticien, l'application sépare les résultats en deux listes selon la présence ou non du champ `id_expert` :

```dart title='details.dart — Filtrage'
List notesClients = notes.where((n) => n['id_expert'] == null).toList();
List notesExperts = notes.where((n) => n['id_expert'] != null).toList();
```

### Recherche en temps réel

La barre de recherche envoie une requête à l'API à chaque modification du champ texte. Un bouton "croix" permet d'effacer la recherche et de revenir à la liste complète :

```dart title='home_screen.dart — Recherche'
onChanged: (value) async {
  if (value.isEmpty) {
    // Recharge la liste complète
    _loadPraticiens();
    return;
  }
  final results = await http.get(
    Uri.parse('http://172.23.48.1/api/praticiens/search?search=$value'),
  );
  setState(() {
    praticiens = json.decode(results.body)['data'];
  });
}
```

---

## UI / UX

| Élément | Description |
|--------|-------------|
| `CustomScrollView + SliverAppBar` | En-tête déroulant sur la page de détail |
| Dégradé de couleurs | Fond AppBar en `LinearGradient` (primaire → secondaire) |
| Cartes Material | Affichage des praticiens dans la liste |
| `TabBar` | Onglets Notes Clients / Notes Experts dans le détail |
| `CircularProgressIndicator` | Indicateur de chargement lors des appels API |
| `SnackBar` | Retours visuels (succès / erreur) lors de la soumission d'une note |
| Écran d'erreur réseau | Affiché si l'API est inaccessible, avec bouton "Réessayer" |
| Icônes Material | `Icons.search_rounded`, `Icons.person_rounded`, `Icons.star_outline_rounded` |

---

## Intégration avec GSBRH

GSBNote fonctionne entièrement comme un **client de l'API GSBRH**. La logique métier (authentification, gestion des praticiens, des notes) est entièrement déléguée au backend Laravel.

Le token JWT obtenu lors de la connexion est transmis dans le header de chaque requête protégée :

```
Authorization: Bearer <token>
```

Seule la soumission d'une note (`POST /api/note/create`) nécessite ce token. La consultation des praticiens et de leurs notes est **publique** et accessible sans connexion.
