# 🎫 Système de gestion des badges QR — Événements mc2i

Outil interne de génération, d'envoi et de scan de QR Codes pour contrôler les présences aux événements d'entreprise. Remplace une solution payante par un assemblage de services Google (gratuits) et d'une page web hébergée sur GitHub Pages.

---

## 1. Vue d'ensemble de l'infrastructure

```
┌─────────────────────────┐         ┌──────────────────────────────┐
│   GOOGLE FORM           │         │   GOOGLE SHEET (la BDD)      │
│   Inscription invités   ├────────►│   Onglets :                  │
└─────────────────────────┘  sync   │   • Invités   (la BDD)       │
                                    │   • Configuration (mail)     │
┌─────────────────────────┐         │   • Réponses au formulaire 1 │
│   APPS SCRIPT           │◄───────►│   • Scans (journal global)   │
│   (lié au Sheet)        │         └──────────▲───────────────────┘
│   • Génère les tokens   │                    │ API Google Sheets
│   • Crée les QR Codes   │                    │ (REST, HTTPS, OAuth)
│   • Envoie les emails   │         ┌──────────┴───────────────────┐
└─────────────────────────┘         │   PAGE DE SCAN (index.html)  │
                                    │   GitHub Pages               │
┌─────────────────────────┐         │   tanguybourhis-lab.github.io│
│   GOOGLE CLOUD CONSOLE  │         │   /qr-code/                  │
│   • API Sheets activée  │◄────────┤   • Caméra live (html5-qrcode)│
│   • ID client OAuth     │  auth   │   • Login Google des hôtes   │
│   (écran de consentement│         │   • Validation + compteurs   │
│    "Interne" mc2i.fr)   │         └──────────────────────────────┘
└─────────────────────────┘
```

### Pourquoi cette architecture ?

| Contrainte rencontrée | Solution retenue |
|---|---|
| Apps Script ne peut pas servir une page avec caméra live (`getUserMedia` bloqué par l'iframe sandbox de Google) | La page de scan est hébergée **hors** d'Apps Script, sur GitHub Pages (HTTPS) |
| La DSI interdit les Web Apps Apps Script anonymes (« Tout le monde ») → impossible d'appeler Apps Script depuis un autre domaine | La page de scan n'appelle **pas** Apps Script : elle parle directement à l'**API Google Sheets**, qui supporte le cross-origin (CORS) |
| L'API Sheets exige une authentification | Chaque hôte/hôtesse se connecte avec son **compte Google mc2i** (popup OAuth) ; la donnée ne sort jamais de l'écosystème Google |

### Les briques

1. **Google Sheet** — la base de données unique.
   - `Invités` : Email (A), Prénom (B), Nom (C), Statut Invitation (D), Token (E), QR envoyé le (F), **Présence (G)** ← horodatage du 1er scan.
   - `Configuration` : modèle du mail d'invitation (sujet + corps, variables `{{PRENOM}}`, `{{QR_CODE}}`…).
   - `Réponses au formulaire 1` : alimenté par le Google Form d'inscription.
   - `Scans` : **créé automatiquement** par la page de scan — journal global de tous les scans (horodatage, type, invité, token, hôte).

2. **Apps Script** (`Code.gs` + `QRLib.gs`, lié au Sheet) — le back-office :
   - synchronisation Form → onglet `Invités` (manuelle ou automatique via trigger) ;
   - génération des tokens (UUID v4, infalsifiables) pour les invités « Acceptée » ;
   - génération des QR Codes (le QR contient **uniquement le token**) et envoi par Gmail ;
   - menu `🎫 Gestion événement` directement dans le Sheet.

3. **Page de scan** (`index.html` sur GitHub Pages) — l'IHM des hôtes :
   - scan caméra en continu (lib `html5-qrcode`) ;
   - validation instantanée contre le Sheet via l'API Google Sheets ;
   - écran 🟢 validé / 🟠 déjà scanné (+ horodatage du 1er scan) / 🔴 refusé, avec bip + vibration ;
   - compteurs globaux temps réel (Inscrits / Validés / Déjà scannés / Refusés) partagés entre tous les téléphones ;
   - historique local des scans, bouton Annuler, garde anti-double-scan et anti-concurrence multi-hôtes.

4. **Google Cloud Console** — l'autorisation :
   - un projet GCP avec l'**API Google Sheets activée** ;
   - un **écran de consentement OAuth** de type **Interne** (réservé aux comptes mc2i.fr) ;
   - un **ID client OAuth « Application Web »** dont les *Origines JavaScript autorisées* contiennent `https://tanguybourhis-lab.github.io`.

### Où vit la configuration ?

En tête du script de `index.html` :

```js
var CLIENT_ID      = '….apps.googleusercontent.com'; // ID client OAuth (Cloud Console)
var SPREADSHEET_ID = '…';                            // ID du Sheet (visible dans son URL)
var SHEET_NAME     = 'Invités';                      // onglet BDD
var COL            = { PRENOM: 1, NOM: 2, TOKEN: 4, PRESENCE: 6 }; // index 0-based
```

### Sécurité

- Les tokens sont des **UUID v4** : non devinables, à usage unique, sans donnée personnelle dans le QR.
- Aucun accès anonyme : toute lecture/écriture passe par le compte Google de l'hôte connecté, qui doit avoir reçu un droit **Éditeur** sur le Sheet.
- Les jetons d'accès OAuth expirent (~1 h) et sont rafraîchis automatiquement.
- L'onglet `Scans` trace chaque opération avec l'email de l'hôte (auditabilité).

---

## 2. Mode opératoire

### A. Préparation (organisateur) — avant l'événement

1. **Collecter les inscriptions** : diffuser le Google Form. Les réponses arrivent dans l'onglet `Réponses au formulaire 1`.
2. Dans le Sheet, menu **🎫 Gestion événement** :
   1. `🔄 Synchroniser les réponses Form` (ou activer la sync auto une fois pour toutes) → remplit/à-jour l'onglet `Invités` ;
   2. `1. Générer les tokens manquants` → crée un token pour chaque invité « Acceptée » ;
   3. *(optionnel)* `⚙️ Créer l'interface de modèle de mail` puis personnaliser le mail dans l'onglet `Configuration` ;
   4. `🔍 Aperçu : compter les invités à traiter` pour vérifier ;
   5. `2. Envoyer les QR Codes aux invités acceptés` → chaque invité reçoit son badge par email, la colonne F est horodatée.
3. **Donner accès aux hôtes** : partager le Google Sheet en **Éditeur** avec chaque hôte/hôtesse (leur compte mc2i).
4. **Remise à zéro avant le jour J** (si des tests ont été faits) :
   - vider les cellules de la colonne **G (Présence)** issues des tests dans `Invités` ;
   - supprimer les lignes de données de l'onglet **`Scans`** (conserver la ligne d'en-tête).

### B. Jour J (hôtes/hôtesses)

1. Sur smartphone, ouvrir **https://tanguybourhis-lab.github.io/qr-code/**.
2. Toucher **« Se connecter avec Google »** → choisir son compte **mc2i** dans la popup.
3. Autoriser la **caméra** quand le navigateur le demande.
4. Scanner les badges :
   - 🟢 **écran vert « Bienvenue ! »** → invité attendu, présence enregistrée (horodatage écrit en colonne G) ;
   - 🟠 **écran orange « Badge déjà scanné »** → badge valide mais déjà utilisé ; l'horodatage du **premier** scan s'affiche ;
   - 🔴 **écran rouge « Badge non reconnu »** → QR absent de la base : refuser l'entrée.
   - L'écran se referme seul après 3 s (ou d'un tap) et la caméra reprend.
5. Repères à l'écran :
   - **Compteurs** (Inscrits / Validés / Déjà scannés / Refusés) : totaux **globaux**, tous téléphones confondus, rafraîchis toutes les 30 s ;
   - **🕒 Historique des scans** : les derniers scans de **ce** téléphone (identité + horodatage) ;
   - **✕ Annuler la vérification** : si une vérification s'éternise (réseau), annule et relance la caméra. Au pire, l'écran se débloque seul après 12 s.
6. Plusieurs hôtes peuvent scanner **en parallèle** : un même badge ne peut être validé qu'une fois (garde anti-concurrence).

### C. Suivi pendant / après l'événement

- L'onglet `Invités`, colonne **G**, donne en direct qui est présent (et depuis quand).
- L'onglet `Scans` est le **journal complet** : chaque tentative de scan, son résultat, l'invité concerné et l'hôte qui a scanné.
- Les compteurs de la page donnent le taux de remplissage en un coup d'œil (Validés / Inscrits).

### D. Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Le badge de version (en-tête de la page) n'est pas le dernier déployé | Cache navigateur / GitHub Pages (~10 min) | Recharger avec `?v=X` dans l'URL, ou navigation privée |
| Popup Google : `origin_mismatch` / `redirect_uri_mismatch` | L'origine GitHub Pages manque dans l'ID client OAuth | Cloud Console → Identifiants → ajouter `https://tanguybourhis-lab.github.io` aux Origines JavaScript |
| Erreur `API 403` au scan | L'hôte n'a pas le droit Éditeur sur le Sheet | Partager le Sheet avec son compte |
| Erreur `API 404` | `SPREADSHEET_ID` erroné dans `index.html` | Vérifier l'ID dans l'URL du Sheet |
| Caméra inaccessible (`NotAllowedError`) | Permission refusée | Réglages du navigateur → autoriser la caméra pour le site |
| Diagnostic approfondi | — | Ajouter **`?debug=1`** à l'URL → panneau de logs à l'écran (mobile) ; sur PC, console F12, préfixe `[SCAN]` |

### E. Maintenance

- **Modifier la page de scan** : éditer `index.html` → repo GitHub `qr-code` → remplacer le fichier → Commit. En ligne en ~1 min (+ cache). Incrémenter le numéro de version affiché dans l'en-tête pour vérifier le déploiement.
- **Nouvel événement** : mettre à jour `EVENT_NAME`, `EVENT_DATE`, `EVENT_LOCATION` dans `Code.gs`, vider les colonnes D à G de `Invités` (ou repartir d'une feuille vierge), purger l'onglet `Scans`.
- **Changer de Sheet** : reporter le nouvel ID dans `SPREADSHEET_ID` (`index.html`) et re-publier la page.
- **Révoquer un hôte** : retirer son droit Éditeur sur le Sheet — son prochain appel API échouera (403).

---

## 3. Fichiers du projet

| Fichier | Rôle | Où il vit |
|---|---|---|
| `Code.gs` | Back-office : sync Form, tokens, envoi des QR par mail, menu Sheet | Projet Apps Script lié au Sheet |
| `QRLib.gs` | Librairie de génération de QR Codes (qrcode-generator, MIT) | Projet Apps Script |
| `index.html` | IHM de scan (caméra, validation, compteurs) | Repo GitHub → GitHub Pages |
| `Scanner.html` | Ancienne IHM servie par Apps Script (mode photo) — conservée à titre de secours | Projet Apps Script (optionnel) |
| `README.md` | Ce document | Repo GitHub / dossier projet |
