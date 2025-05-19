# Documentation de l'API - Kit le nid

Ce document décrit l'ensemble des endpoints de l'application ainsi que leur utilisation.

## Sommaire

- [1. Meetings](#1-meetings)
- [2. Recherche Géographique](#2-recherche-géographique)
- [3. Gestion des Favoris](#3-gestion-des-favoris)
- [4. Informations sur les Transports](#4-informations-sur-les-transports)
- [5. Authentification via NextAuth](#5-authentification-via-nextauth)
- [6. Enregistrement d'un Nouvel Utilisateur](#6-enregistrement-dun-nouvel-utilisateur)
- [7. Gestion des Utilisateurs](#7-gestion-des-utilisateurs)
- [8. Requêtes vers l'API Wordpress (Blog et Lancements)](#8-requêtes-vers-lapi-wordpress-blog-et-lancements)

---

## 1. Meetings

**URL :** `/api/meetings`

**Méthode :** `POST`

**Description :**  
Cet endpoint permet de créer un rendez-vous. Avant la création, le système vérifie que le créneau n'est pas déjà réservé. En cas de succès, un email de confirmation (contenant un fichier ICS pour l'intégration dans le calendrier) est envoyé aux utilisateurs.

**Corps de la requête (JSON) :**

```json
{
"name": "Nom du prospect",
"lastname": "Prénom du prospect",
"emailadress": "exemple@domaine.com",
"phone": "0123456789",
"date": "2023-12-31T09:00:00Z",
"time": "10:00-11:00"
},
```

**Réponses possibles :**

- **Succès (200) :**

  ```json
  {
    "success": true
  }
  ```

- **Erreur – Créneau réservé (400) :**
  ```json
  {
    "error": "Ce créneau est déjà réservé."
  }
  ```

**Notes :**

- La validation des données est effectuée à l'aide de Zod.
- En cas d'erreur lors de la création du rendez-vous, l'API retourne un message explicite pour guider l'utilisateur.

---

## 2. Recherche Géographique

### Recherche géographique officielle via l'API https://geo.api.gouv.fr

**URL de base :** `https://geo.api.gouv.fr`

**Méthode :** `GET`

**Description :**  
Cette API officielle permet d'obtenir des informations détaillées sur les communes de France. Selon le type de recherche, la requête peut s'effectuer par nom de commune ou par code postal.

**Paramètres et exemples d'URL :**

- **Recherche par code postal (code à 5 chiffres)** :  
  Requête :
  ```
  GET https://geo.api.gouv.fr/communes?codePostal=75001&fields=nom,code,codesPostaux,centre,departement,population,contour
  ```
- **Recherche par nom de commune** :  
  Requête :
  ```
  GET https://geo.api.gouv.fr/communes?nom=paris&fields=nom,code,codesPostaux,centre,departement,population,contour&boost=population&limit=5
  ```

**Exemple de réponse (Succès 200) :**

```json
[
  {
    "nom": "Paris",
    "code": "75056",
    "codesPostaux": ["75001"],
    "centre": {
      "type": "Point",
      "coordinates": [2.3522, 48.8566]
    },
    "departement": {
      "nom": "Paris",
      "code": "75"
    },
    "population": 2148000,
    "contour": {
      "type": "Polygon",
      "coordinates": [
        /* coordonnées géométriques */
      ]
    }
  }
]
```

**Notes :**

- Pour plus d'informations sur les champs et les fonctionnalités, consultez la [documentation officielle de l'API Géolocalisation](https://geo.api.gouv.fr/documentation).

---

### Recherche géographique locale via l'API interne

**URL :** `/api/geo`

**Méthode :** `GET`

**Description :**  
Cet endpoint, développé en interne, permet d'effectuer une recherche géographique basée sur un jeu de données local (GeoMaps). Il s'adresse principalement à la recherche de départements ou de régions selon la requête de l'utilisateur.

**Paramètres acceptés :**

- `q` : Chaîne de recherche (minimum 2 caractères)
- `type` : `"all"` (par défaut), `"department"` ou `"region"`

**Exemple d'URL :**

```
/api/geo?q=paris&type=department
```

**Exemple de réponse (Succès 200) :**

```json
[
  {
    "type": "department",
    "name": "Paris",
    "code": "75",
    "geometry": {
      /* données géométriques locales */
    }
  }
]
```

**Notes :**

- Les résultats sont limités aux 10 premiers éléments.
- Cette recherche, implémentée dans `/api/geo`, exploite le fichier GeoMaps afin de renvoyer des informations sur les départements et les régions.

---

## 3. Gestion des Favoris

### Ajouter un Favori

**URL :** `/api/favorites/{propertyId}`

**Méthode :** `POST`

**Description :**  
Ajoute un bien immobilier (identifié par `propertyId`) aux favoris de l'utilisateur.

**Corps de la requête (JSON) :**

```json
{
  "userId": "identifiant_de_l_utilisateur"
}
```

**Réponse (Succès 200) :**

```json
{
  "id": "identifiant_de_l_utilisateur",
  "firstName": "Prénom",
  "lastName": "Nom",
  "email": "exemple@domaine.com",
  "image": "url_image",
  "role": "role_utilisateur",
  "favoriteIds": ["liste", "des", "propertyIds", "mis à jour"]
}
```

### Retirer un Favori

**URL :** `/api/favorites/{propertyId}`

**Méthode :** `DELETE`

**Description :**  
Retire un bien immobilier des favoris de l'utilisateur.

**Corps de la requête (JSON) :**

```json
{
  "userId": "identifiant_de_l_utilisateur"
}
```

**Réponse (Succès 200) :**

```json
{
  "id": "identifiant_de_l_utilisateur",
  "firstName": "Prénom",
  "lastName": "Nom",
  "email": "exemple@domaine.com",
  "image": "url_image",
  "role": "role_utilisateur",
  "favoriteIds": ["liste", "des", "propertyIds", "après suppression"]
}
```

---

## 4. Informations sur les Transports

**URL :** `/api/transport`

**Méthode :** `GET`

**Description :**  
Cet endpoint récupère les informations sur les transports à proximité en fonction des coordonnées géographiques fournies. Le service a été migré d'une implémentation MBI vers un service dédié pour améliorer la maintenabilité et les performances.

**Paramètres de requête :**

- `lon` : Longitude (ex. `2.35`)
- `lat` : Latitude (ex. `48.85`)

**Exemple d'URL :**  
`/api/transport?lon=2.35&lat=48.85`

**Exemple de réponse (Succès 200) :**

```json
{
  "data": [
    {
      "id": "transport_1",
      "type": "metro",
      "line": "4",
      "name": "Saint-Michel",
      "distance": 450,
      "coordinates": {
        "lat": 48.85,
        "lon": 2.35
      }
    },
    {
      "id": "transport_2",
      "type": "bus",
      "line": "96",
      "name": "Saint-Germain",
      "distance": 200,
      "coordinates": {
        "lat": 48.853,
        "lon": 2.348
      }
    }
  ]
}
```

**Remarque :**  
Les headers CORS sont configurés pour autoriser l'accès depuis n'importe quelle origine. Le service utilise désormais transportService.ts pour une meilleure maintenabilité.

---

## 5. Authentification via NextAuth

**URL :** `/api/auth/[...nextauth]`

**Méthodes :** `GET`, `POST`

**Description :**  
Cet endpoint gère l'ensemble des flux d'authentification (connexion, déconnexion, etc.) à l'aide de NextAuth. La configuration détaillée est définie dans `authOptions`.

---

## 6. Enregistrement d'un Nouvel Utilisateur

**URL :** `/api/auth/register`

**Méthode :** `POST`

**Description :**  
Permet d'enregistrer un nouvel utilisateur. L'endpoint vérifie l'existence d'un utilisateur avec le même email, hache le mot de passe et crée l'utilisateur dans la base de données. Un email de notification est ensuite envoyé.

**Corps de la requête (JSON) :**

```json
{
  "firstName": "Prénom",
  "lastName": "Nom",
  "phone": "0123456789",
  "email": "exemple@domaine.com",
  "password": "motdepasse",
  "role": "role_utilisateur",
  "dataExploitation": "informations_supplémentaires"
}
```

**Réponse (Succès 200) :**

```json
{
  "id": "identifiant_utilisateur",
  "firstName": "Prénom",
  "lastName": "Nom",
  "phone": "0123456789",
  "email": "exemple@domaine.com",
  "role": "role_utilisateur",
  "dataExploitation": "informations_supplémentaires"
}
```

**Réponse en cas d'erreur (400) :**

```json
{
  "error": "L'utilisateur existe déjà"
}
```

---

## 7. Gestion des Utilisateurs

**URL :** `/api/user/{id}`

**Méthodes :** `GET`, `PATCH`, `DELETE`

### Récupérer un Utilisateur

**Méthode :** `GET`

**Description :**  
Permet de récupérer les informations d'un utilisateur à partir de son identifiant.

**Exemple d'URL :**  
`/api/user/12345`

**Exemple de réponse (Succès 200) :**

```json
{
  "id": "12345",
  "firstName": "Prénom",
  "lastName": "Nom",
  "email": "exemple@domaine.com",
  "image": "url_image",
  "role": "role_utilisateur",
  "favoriteIds": ["liste", "des", "propertyIds"]
}
```

#### Mettre à Jour un Utilisateur

**Méthode :** `PATCH`

**Description :**  
Permet de mettre à jour les informations d'un utilisateur. Le corps de la requête doit contenir les champs à modifier.

**Corps de la requête (exemple) :**

```json
{
  "firstName": "NouveauPrénom",
  "email": "nouveau@domaine.com"
}
```

**Réponse (Succès 200) :**

```json
{
  "id": "12345",
  "firstName": "NouveauPrénom",
  "lastName": "Nom",
  "email": "nouveau@domaine.com",
  "image": "url_image",
  "role": "role_utilisateur",
  "favoriteIds": ["liste", "des", "propertyIds"]
}
```

#### Supprimer un Utilisateur

**Méthode :** `DELETE`

**Description :**  
Permet de supprimer un utilisateur par son identifiant.

**Exemple d'URL :**  
`/api/user/12345`

**Réponse (Succès 200) :**

```json
{
  "message": "Utilisateur supprimé avec succès"
}
```

---

## 8. Requêtes vers l'API Wordpress (Blog et Lancements)

Les endpoints suivants permettent de récupérer les données du blog et de la section "Lancements" via l'API Wordpress.

### Articles du Blog

**URL de base :** `https://blog.kitlenid.fr/wp-json/wp/v2/posts`

**Méthode :** `GET`

**Paramètres de requête (exemple) :**

- `_fields` : Liste des champs à récupérer (exemple : "id, date, title, content, excerpt, ...")
- `per_page` : Nombre d'articles par page (exemple : 10)
- `page` : Numéro de page (exemple : 1)

**Exemple d'URL :**  
`https://blog.kitlenid.fr/wp-json/wp/v2/posts?per_page=10&page=1`

**Exemple de réponse :**

```json
[
{
"id": 123,
"date": "2023-01-01T12:00:00",
"title": { "rendered": "Titre de l'article" },
"content": { "rendered": "<p>Contenu de l'article</p>" },
"excerpt": { "rendered": "Résumé de l'article" }
},
...
]
```

### Lancements

**URL de base :** `https://blog.kitlenid.fr/wp-json/wp/v2/lancements`

**Méthode :** `GET`

**Paramètres de requête (exemple) :**

- `_fields` : Liste des champs à récupérer, incluant notamment les informations spécifiques aux lancements (par exemple : "id, date, title, content, categories, tags")
- `per_page` : Nombre de lancements par page (exemple : 10)
- `page` : Numéro de page (exemple : 1)

**Exemple d'URL :**  
`https://blog.kitlenid.fr/wp-json/wp/v2/lancements?per_page=10&page=1`

**Exemple de réponse :**

```json
[
{
"id": 456,
"date": "2023-02-01T10:00:00",
"title": { "rendered": "Titre du lancement" },
"content": { "rendered": "<p>Contenu du lancement</p>" },
"categories": [1, 2],
"tags": ["Nouveau", "Promotion"]
},
...
]
```

---

### Recherche de Propriétés

**URL :** `/api/properties`

**Méthode :** `GET`

**Description :**  
Cet endpoint permet de rechercher des propriétés selon différents critères géographiques. La logique de recherche a été optimisée pour fournir des résultats plus pertinents :

- Pour les recherches par ville/code postal :

  - Affiche tous les résultats exacts dans la ville
  - Combine avec les 6 propriétés les plus proches dans un rayon de 60km (hors de la ville exacte)
  - Retourne une seule page de résultats combinés

- Pour les recherches par région/département :
  - Affiche les 6 premiers résultats trouvés dans la zone
  - Utilise la pagination standard

**Paramètres de requête :**

- `city` : Nom de la ville ou code postal
- `region` : Nom de la région ou département
- `coordinates` : Coordonnées géographiques [longitude, latitude]
- `radius` : Rayon de recherche en kilomètres (par défaut : 60)
- `page` : Numéro de page (par défaut : 1)
- `limit` : Nombre de résultats par page (par défaut : 6)

**Exemple de réponse (Succès 200) :**

```json
{
  "data": [
    {
      "id": "property_1",
      "title": "Appartement T3",
      "location": {
        "type": "Point",
        "coordinates": [2.3522, 48.8566]
      },
      "price": 250000,
      "surface": 75,
      "rooms": 3
    }
  ],
  "metadata": {
    "totalResults": 10,
    "page": 1,
    "hasNextPage": true,
    "totalPages": 2
  }
}
```

**Notes :**

- L'endpoint utilise un index `2dsphere` sur le champ `location` pour les recherches géospatiales.
- La création de l'index doit être effectuée manuellement via `mongosh` car `prisma db push` ne crée pas correctement l'index (jusqu'à Prisma v6.6.0).
- Les résultats sont triés par distance pour les recherches géographiques.

---

Ce document regroupe tous les endpoints principaux de l'API de l'application Kit le nid ainsi que les requêtes vers l'API Wordpress et l'API de géolocalisation publique.
Il est recommandé de le maintenir à jour au fur et à mesure de l'évolution de l'application.
