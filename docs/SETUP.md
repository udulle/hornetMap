# Guide d'installation — Carte des Frelons

## Vue d'ensemble

L'application se compose de 3 éléments :

| Élément | Rôle |
|---------|------|
| **Google Sheets** | Base de données des observations (partagée avec les observateurs) |
| **Google Forms** | Formulaire de saisie pour les observateurs terrain |
| **Page web** (`web/index.html`) | Carte interactive Leaflet + tableau, hébergée sur GitHub Pages |

Les observateurs ajoutent des données via le formulaire ou directement dans le tableur. La carte se met à jour automatiquement à chaque chargement de page (pas de rebuild nécessaire).

---


## 0. Test local (Windows)

La page charge des fichiers GeoJSON via `fetch()`, ce qui ne fonctionne pas en ouvrant directement `index.html` depuis l'explorateur (`file://`). Il faut un petit serveur HTTP local.

### Option A : Python (si installé)

1. Ouvrir un terminal (PowerShell ou Invite de commandes)
2. Se placer dans le dossier `web` :
   ```
   cd C:\chemin\vers\hornetMap\web
   ```
3. Lancer le serveur :
   ```
   python -m http.server 8080
   ```
4. Ouvrir le navigateur à l'adresse : **http://localhost:8080**

> Pour arrêter le serveur : `Ctrl+C` dans le terminal.

### Option B : Extension VS Code « Live Server »

1. Installer l'extension **Live Server** dans VS Code (identifiant : `ritwickdey.LiveServer`)
2. Ouvrir le dossier `web/` dans VS Code
3. Clic droit sur `index.html` → **Open with Live Server**
4. Le navigateur s'ouvre automatiquement (en général sur `http://127.0.0.1:5500`)

### Option C : Node.js (si installé)

1. Ouvrir un terminal dans le dossier `web` :
   ```
   cd C:\chemin\vers\hornetMap\web
   npx serve .
   ```
2. Ouvrir l'URL affichée (en général **http://localhost:3000**)

### Vérifications

- La carte s'affiche avec les tuiles CartoDB
- Les marqueurs apparaissent (points du Google Sheet)
- Les couches GeoJSON (Communes, FA Zone Nyon) sont disponibles dans le contrôle de couches
- La colonne Commune est remplie automatiquement dans le tableau

---

## 1. Google Sheets — Base de données

### 1.1 Créer le tableur

1. Ouvrir [Google Sheets](https://sheets.google.com) → **Nouveau tableur**
2. Nommer le tableur (ex: `Observations Frelons — SAN`)
3. Créer les colonnes suivantes **exactement** en ligne 1 :

| A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|
| Name | Category | Latitude | Longitude | Note | Date | Active |

> **Important** : Les noms de colonnes doivent correspondre exactement (majuscule initiale). Le code JavaScript lit ces en-têtes tels quels.

### 1.2 Validation des données (recommandé)

Pour éviter les erreurs de saisie :

- **Colonne B (Category)** : Sélectionner B2:B1000 → menu **Données → Validation des données → Ajouter une règle** → **Liste déroulante** avec les valeurs :
  - `Observation`
  - `Nid Primaire`
  - `Nid Secondaire`
- **Colonne C (Latitude)** : Validation → **Le nombre est compris entre** → `45.5` et `47.5`
- **Colonne D (Longitude)** : Validation → **Le nombre est compris entre** → `5.5` et `7.5`
- **Colonne G (Active)** : Liste déroulante → `TRUE`, `FALSE`

### 1.3 Partager le tableur

1. Cliquer **Partager** (bouton vert en haut à droite)
2. Sous **Accès général**, choisir **Tous les utilisateurs disposant du lien** → **Lecteur**
3. Pour les observateurs autorisés à modifier directement le tableur : ajouter leurs adresses email avec le rôle **Éditeur**

### 1.4 Obtenir l'URL CSV

L'URL de flux CSV est construite à partir de l'ID du tableur :

```
https://docs.google.com/spreadsheets/d/{SHEET_ID}/gviz/tq?tqx=out:csv&gid=0
```

Pour trouver le `SHEET_ID` : ouvrir le tableur dans le navigateur, l'URL ressemble à :
```
https://docs.google.com/spreadsheets/d/1X8OCgibaeocZmdFeJl5Qtj8z4Jh_AKj_4AeN2aIS7co/edit
                                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                        ← c'est le SHEET_ID
```

> **Note** : `gid=0` désigne le premier onglet. Si les données sont dans un autre onglet, ajuster le `gid` (visible dans l'URL quand on clique sur l'onglet).

### 1.5 Configurer dans la page web

Ouvrir `web/index.html`, trouver la section `CONFIG` et coller l'URL CSV :

```javascript
var CONFIG = {
  sheetCsvUrl: 'https://docs.google.com/spreadsheets/d/VOTRE_SHEET_ID/gviz/tq?tqx=out:csv&gid=0',
  ...
};
```

---

## 2. Google Forms — Formulaire de saisie

### 2.1 Créer le formulaire

1. Ouvrir [Google Forms](https://forms.google.com) → **Formulaire vierge**
2. Nommer le formulaire (ex: `Signalement Frelon à pattes jaunes`)
3. Ajouter les questions suivantes :

| # | Question | Type | Options / Validation |
|---|----------|------|----------------------|
| 1 | **Nom** | Réponse courte | Obligatoire |
| 2 | **Catégorie** | Liste déroulante | `Observation`, `Nid Primaire`, `Nid Secondaire` — Obligatoire |
| 3 | **Latitude** | Réponse courte | Obligatoire. Description : `ex: 46.3775` |
| 4 | **Longitude** | Réponse courte | Obligatoire. Description : `ex: 6.2310` |
| 5 | **Note** | Réponse courte | Optionnel. Description : `Lieu-dit, détails` |
| 6 | **Date** | Date | Obligatoire |

### 2.2 Lier le formulaire au tableur

**Option A — Depuis le formulaire :**
1. Dans Google Forms, onglet **Réponses**
2. Cliquer **Associer à Sheets** (icône tableur vert)
3. Choisir **Sélectionner un tableur existant** → sélectionner le tableur créé à l'étape 1
4. Un nouvel onglet nommé `Réponses au formulaire 1` est créé automatiquement

**Option B — Depuis le tableur :**
1. Dans Google Sheets, menu **Outils → Créer un formulaire**
2. Le formulaire est créé et lié automatiquement

### 2.3 Harmoniser les colonnes

Le formulaire crée un onglet séparé avec une colonne `Horodateur` en plus. Deux approches :

**Approche simple (recommandée)** : Pointer le `gid` dans `CONFIG.sheetCsvUrl` vers l'onglet des réponses du formulaire.

1. Cliquer sur l'onglet `Réponses au formulaire 1` dans le tableur
2. Lire le `gid` dans l'URL du navigateur (ex: `gid=123456789`)
3. Mettre à jour `CONFIG.sheetCsvUrl` avec ce `gid`
4. Renommer les colonnes du formulaire pour qu'elles correspondent exactement : `Name`, `Category`, `Latitude`, `Longitude`, `Note`, `Date`

> **Attention** : Google Forms utilise la question comme nom de colonne. Pour que le code JavaScript fonctionne, les questions du formulaire doivent correspondre exactement aux noms attendus (`Name`, `Category`, `Latitude`, `Longitude`, `Note`, `Date`), **ou** il faut renommer les en-têtes dans l'onglet de réponses après coup (les nouvelles réponses respecteront les noms renommés).

**Approche alternative** : Garder les données manuelles dans l'onglet 1 (`gid=0`) et les réponses du formulaire dans l'onglet 2. Copier-coller périodiquement les nouvelles lignes de l'onglet formulaire vers l'onglet principal.

### 2.4 Obtenir l'URL du formulaire

1. Dans Google Forms, cliquer **Envoyer** (bouton en haut à droite)
2. Onglet **Lien** (icône chaîne)
3. Copier l'URL (optionnel : cocher **Raccourcir l'URL**)

### 2.5 Configurer dans la page web

L'URL du formulaire est déjà dans `web/index.html` sur le bouton **+ Signaler une observation**. Pour la modifier :

```html
<a href="https://docs.google.com/forms/d/e/VOTRE_FORM_ID/viewform" target="_blank" ...>
```

---

## 3. Hébergement — GitHub Pages

### 3.1 Prérequis

- Un compte [GitHub](https://github.com) (gratuit)
- Git installé localement (`sudo apt install git` ou [télécharger](https://git-scm.com))

### 3.2 Créer le dépôt et pousser le code

```bash
cd ~/hornetMap
git init
git checkout -b main

# Fichiers à versionner
cat > .gitignore << 'EOF'
.Rhistory
.RData
.Rproj.user/
couches_gis.gpkg
*.html
!web/**
EOF

git add web/ .gitignore AGENTS.md
git commit -m "Initial commit: hornet map web app"
```

Créer un dépôt sur GitHub (https://github.com/new), puis :

```bash
git remote add origin https://github.com/udulle/hornetMap.git
git push -u origin main
```

### 3.3 Activer GitHub Pages

1. Sur GitHub, aller dans **Settings** du dépôt
2. Section **Pages** (menu de gauche)
3. Source : **Deploy from a branch**
4. Branch : `main`, dossier : `/web`  
   *(Note : GitHub Pages ne supporte que `/` ou `/docs`. Si `/web` n'est pas proposé, renommer le dossier en `docs/` ou configurer une GitHub Action)*
5. Cliquer **Save**

**Alternative avec `/docs`** :
```bash
mv web docs
git add -A && git commit -m "Rename web/ to docs/ for GitHub Pages"
git push
```
Puis choisir `/docs` comme dossier dans les paramètres Pages.

### 3.4 URL de la page

Après quelques minutes, la carte est accessible à :
```
https://VOTRE_USER.github.io/hornetMap/
```

### 3.5 Domaine personnalisé (optionnel)

Pour utiliser un sous-domaine comme `carte.apiculture-nyon.ch` :

1. Dans GitHub Pages **Settings → Custom domain** : entrer `carte.apiculture-nyon.ch`
2. Chez le registrar DNS du domaine : ajouter un enregistrement **CNAME** :
   ```
   carte.apiculture-nyon.ch  →  VOTRE_USER.github.io
   ```
3. Attendre la propagation DNS (~1h), puis cocher **Enforce HTTPS**

---

## 4. Intégration Wix — apiculture-nyon.ch

### Option A : Lien externe (fonctionne avec tout plan Wix)

1. Dans l'éditeur Wix, ouvrir le **menu de navigation**
2. Ajouter un élément de menu : **Lien externe**
3. URL : `https://VOTRE_USER.github.io/hornetMap/`
4. Texte : `Carte des Frelons`
5. Cocher **Ouvrir dans un nouvel onglet**

### Option B : Iframe intégré (nécessite Wix Premium)

1. Dans l'éditeur Wix, cliquer **Ajouter des éléments** → **Code intégré** → **Intégrer un site**
2. Cliquer **Entrer l'adresse du site web**
3. Coller l'URL : `https://VOTRE_USER.github.io/hornetMap/`
4. Redimensionner l'iframe (largeur 100%, hauteur ~800px recommandé)
5. Publier le site

> **Note** : L'iframe n'est pas responsive dans Wix. Tester sur mobile et ajuster si nécessaire.

---

## 5. Maintenance

### Ajouter des observations

| Méthode | Qui | Comment |
|---------|-----|---------|
| Formulaire Google | Observateurs terrain | Cliquer **+ Signaler une observation** sur la carte |
| Tableur Google | Administrateurs | Ouvrir le tableur, ajouter une ligne |

Les nouvelles données apparaissent sur la carte au prochain chargement de page (rafraîchir ou F5).

### Modifier la page web

1. Modifier `web/index.html` (ou `docs/index.html`)
2. Commit + push :
   ```bash
   git add -A && git commit -m "Description du changement" && git push
   ```
3. GitHub Pages se met à jour automatiquement (~1 min)

### Ajouter une nouvelle catégorie

1. Dans le tableur : ajouter la valeur dans la validation de la colonne Category
2. Dans le formulaire : ajouter l'option dans la question Catégorie
3. Dans `web/index.html` : ajouter la couleur dans `CONFIG.palette` :
   ```javascript
   palette: {
     'Observation':      '#e74c3c',
     'Nid Primaire':     '#3498db',
     'Nid Secondaire':   '#2ecc71',
     'Nouvelle Catégorie': '#ff9800'   // ← ajouter ici
   }
   ```

### Mettre à jour les couches GIS

Si les limites communales ou du district changent :

```bash
cd ~/hornetMap
ogr2ogr -f GeoJSON web/data/district_nyon.geojson couches_gis.gpkg district_nyon
ogr2ogr -f GeoJSON web/data/communes_nyon.geojson couches_gis.gpkg communes_nyon
git add web/data/ && git commit -m "Update GIS layers" && git push
```
