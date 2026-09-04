# AFME Paris 13 — Site de formation CAPEPS

Site web de l'Association de Formation aux Métiers de l'Enseignement (Paris 13),
dédié à la préparation au CAPEPS (filières Licence et Master).

---

## Ce que fait le site

- **Page de connexion** avec les deux calendriers Google intégrés (Licence / Master)
- **Espace de cours** organisé par filière, épreuve et thème — chaque page a sa propre liste de documents téléchargeables
- **Fil d'actualités** sur l'accueil : l'équipe peut publier texte, image et/ou document combinés
- **Devoirs** : calendrier de travail par filière (Licence / Master), mis à jour par les formateurs
- **Dépôt de fichiers** sur chaque page, avec sélection explicite de la page de destination
- **Panneau d'administration** : validation des inscriptions, modification des rôles, déplacement de documents mal rangés, ouverture de pages au dépôt par les adhérents
- **Thèmes visuels par rôle** : rouge pour l'administrateur, bleu pour les formateurs, jaune pour les adhérents

---

## Architecture technique

| Élément | Choix |
|---|---|
| Livraison | Un seul fichier `index.html` autonome (CSS et JS inlinés) |
| Base de données | Cloud Firestore (Firebase) |
| Stockage fichiers | Firebase Storage |
| Authentification | Firebase Authentication (email / mot de passe) |
| Hébergement suggéré | GitHub Pages (gratuit) |
| Générateur | `build.py` (Python 3, aucune dépendance externe) |

---

## Rôles et droits

| Rôle | Accès | Dépôt | Devoirs | Administration |
|---|---|---|---|---|
| **Administrateur** | Tout | Toutes les pages | Lecture + ajout | Complète |
| **Formateur** | Tout | Toutes les pages | Lecture + ajout | Non |
| **Adhérent** | Tout le contenu | Pages autorisées uniquement | Lecture | Non |
| **En attente** | Accueil + Contact | Non | Non | Non |
| **Non connecté** | Accueil + Contact | Non | Non | Non |

Les pages autorisées pour un adhérent sont soit ouvertes à tous les adhérents (via Administration → Pages ouvertes), soit attribuées nominativement à un compte précis (via Administration → Modifier un compte).

---

## Configuration Firebase

### 1. Créer le projet

Aller sur [console.firebase.google.com](https://console.firebase.google.com) et créer un projet.

### 2. Activer Authentication

Authentication → Sign-in method → **E-mail/Mot de passe** → activer → Enregistrer.

### 3. Créer Firestore

Firestore Database → Créer la base → choisir une région (ex. `europe-west1`) → démarrer en mode production.

### 4. Activer Storage

Storage → Commencer → choisir la même région que Firestore → terminer la configuration.
> ⚠️ Storage nécessite le **forfait Blaze** (paiement à l'usage). Le quota gratuit inclus couvre largement un usage associatif normal (5 Go de stockage, 1 Go/jour de téléchargement). Configurer une alerte de budget à 1 € pour être prévenu en cas de dépassement.

### 5. Règles Firestore

Firestore → onglet **"Règles"** → coller et publier :

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function getProfile() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data;
    }
    function isSignedIn() { return request.auth != null; }
    function isAdmin() { return isSignedIn() && getProfile().role == "admin"; }
    function isStaff() {
      return isSignedIn() && (getProfile().role == "formateur" || getProfile().role == "admin");
    }
    function isAdherent() { return isSignedIn() && getProfile().role == "adherent"; }
    function pageOpen(cat) {
      return exists(/databases/$(database)/documents/pagePermissions/$(cat))
        && get(/databases/$(database)/documents/pagePermissions/$(cat)).data.studentUploadOpen == true;
    }
    function pagePersonal(cat) { return cat in getProfile().uploadPages; }

    match /documents/{docId} {
      allow read: if isSignedIn();
      allow create: if isSignedIn()
        && request.resource.data.uploadedByUid == request.auth.uid
        && (isStaff() || (isAdherent()
             && (pageOpen(request.resource.data.category) || pagePersonal(request.resource.data.category))));
      allow update: if isAdmin();
      allow delete: if isSignedIn()
        && (resource.data.uploadedByUid == request.auth.uid || isAdmin());
    }

    match /actualites/{id} {
      allow read: if isSignedIn();
      allow create: if isStaff();
      allow update, delete: if isAdmin()
        || (isSignedIn() && resource.data.createdByUid == request.auth.uid);
    }

    match /assignments/{id} {
      allow read: if isSignedIn();
      allow create: if isStaff();
      allow update, delete: if isAdmin();
    }

    match /users/{uid} {
      allow read: if isSignedIn() && (request.auth.uid == uid || isAdmin());
      allow create: if isSignedIn() && request.auth.uid == uid
        && request.resource.data.role == "pending";
      allow update: if isAdmin();
    }

    match /pagePermissions/{cat} {
      allow read: if isSignedIn();
      allow write: if isAdmin();
    }
  }
}
```

### 6. Règles Storage

Storage → onglet **"Rules"** → coller et publier :

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /documents/{category}/{fileName} {
      allow read: if true;
      allow write: if request.auth != null
                   && request.resource.size < 25 * 1024 * 1024;
    }
    match /actualites/{folder}/{fileName} {
      allow read: if true;
      allow write: if request.auth != null
                   && request.resource.size < 25 * 1024 * 1024;
    }
  }
}
```

### 7. Récupérer la config SDK

Paramètres du projet (icône ⚙️) → "Vos applications" → application web → copier l'objet `firebaseConfig`.
Dans `build.py`, coller les valeurs dans le dictionnaire `FIREBASE` en haut du fichier, puis relancer `python3 build.py`.

---

## Premiers comptes

### Créer les comptes dans Firebase

Authentication → Users → **Add user** pour chaque personne, avec son email et un mot de passe (6 caractères minimum).

### Attribuer le premier administrateur

1. Se connecter sur le site avec le compte admin (le profil est créé en statut `pending`)
2. Dans Firestore → collection `users` → ouvrir le document de ce compte
3. Modifier le champ `role` : remplacer `pending` par `admin`

C'est la **seule intervention manuelle** dans Firestore. Ensuite tout se gère depuis le panneau Administration du site.

### Valider les inscriptions suivantes

Depuis le site connecté en admin → **Administration → "Comptes en attente"** → bouton "✓ Formateur" ou "✓ Adhérent" selon le cas.

### Comptes de test

| Rôle | Email suggéré | Interface |
|---|---|---|
| Administrateur | `admin@afme-paris13.fr` | Rouge |
| Formateur | `formateur@afme-paris13.fr` | Bleu |
| Adhérent | `adherent@afme-paris13.fr` | Jaune |

Mot de passe suggéré pour les tests : `Test1999` (à changer en production).

---

## Intégrer les calendriers Google

1. Aller sur [calendar.google.com](https://calendar.google.com)
2. Clic droit sur le calendrier voulu → **Paramètres et partage**
3. Section **"Intégrer le calendrier"** → copier l'URL embed
   (format : `https://calendar.google.com/calendar/embed?src=XXX&ctz=Europe%2FParis`)
4. Dans `index.html`, chercher les deux attributs `data-cal-src` et remplacer
   `REPLACE_LICENCE_CALENDAR_EMBED_URL` et `REPLACE_MASTER_CALENDAR_EMBED_URL`
   par les URLs copiées
5. Dans les paramètres du calendrier → **"Autorisations d'accès"** → cocher
   **"Rendre disponible pour tout le monde"** pour que les membres puissent voir les événements

---

## Ajouter ou modifier une page

Toute l'arborescence du site est définie dans le tableau `TREE` en haut du fichier `build.py`.

**Ajouter une page** :

```python
{"title": "Ma nouvelle page", "slug": "formation-master/ma-nouvelle-page"}
```

**Ajouter une rubrique avec sous-pages** :

```python
{
    "title": "Nouvelle rubrique",
    "slug": "formation-master/nouvelle-rubrique",
    "children": [
        {"title": "Sous-page 1", "slug": "formation-master/nouvelle-rubrique/sous-page-1"},
        {"title": "Sous-page 2", "slug": "formation-master/nouvelle-rubrique/sous-page-2"},
    ]
}
```

Puis relancer :

```bash
python3 build.py
```

Le fichier `index.html` est régénéré. Le contenu pédagogique réel (cours, fiches) ne vit pas dans le HTML mais dans Firebase — il n'est donc pas écrasé.

---

## Hébergement sur GitHub Pages

1. Créer un repo GitHub (ex. `afme-paris13`)
2. Déposer `index.html` à la racine (renommé depuis `afme-paris13-site.html` si besoin)
3. Repo → **Settings → Pages** → Source = branche `main`, dossier `/ (root)` → Save
4. Le site est disponible sur `https://TON_PSEUDO.github.io/afme-paris13/`
5. Dans Firebase → **Authentication → Domaines autorisés** → ajouter `TON_PSEUDO.github.io`
   (sinon la connexion sera refusée depuis ce domaine)

Pour un domaine personnalisé (`afmeparis13.fr`) : configurer un enregistrement DNS `CNAME` pointant vers `TON_PSEUDO.github.io`, puis l'ajouter dans GitHub Pages Settings → Custom domain et dans Firebase → Authentication → Domaines autorisés.

---

## Collections Firestore créées automatiquement

| Collection | Contenu |
|---|---|
| `users` | Profils utilisateurs (email, rôle, pages autorisées) |
| `documents` | Métadonnées de chaque fichier déposé (nom, URL, catégorie, auteur) |
| `actualites` | Publications du fil d'actualités (titre, texte, imageUrl, documentUrl) |
| `assignments` | Devoirs (filière, date, intitulé, détail) |
| `pagePermissions` | Pages ouvertes au dépôt par tous les adhérents |

---

## Fichiers du projet

```
afme-paris13/
├── index.html          ← site complet (généré, ne pas éditer à la main)
├── build.py            ← générateur du site (éditer TREE ici pour ajouter des pages)
└── README.md           ← ce fichier
```

---

## Notes importantes

- **Ne jamais éditer `index.html` à la main** pour ajouter des pages ou rubriques — elles seraient écrasées à la prochaine régénération. Utiliser uniquement `build.py`.
- Le contenu pédagogique (cours, fiches, copies) est stocké dans Firebase, pas dans le HTML. Il est donc préservé entre chaque régénération.
- **Sécurité** : la vérification des droits de dépôt est double — côté JavaScript (UX) et côté règles Firestore (protection réelle). La couche Storage n'est protégée que par "être connecté" ; pour une sécurité maximale, migrer vers des Cloud Functions Firebase en cas de besoin.
- Le mot de passe minimum est de **6 caractères** (limite Firebase Authentication).
