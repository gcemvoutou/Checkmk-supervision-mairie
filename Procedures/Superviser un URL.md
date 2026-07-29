# Procédure : Supervision de l'URL Ligeo Gestion dans Checkmk

Cette procédure détaille les étapes pour déclarer et superviser l'URL de l'application **Ligeo Gestion** dans Checkmk, conformément aux consignes du service informatique.

---

## 📌 Informations sur le service

* **Application :** Ligeo (Module Gestion)
* **Éditeur :** Boscop
* **Type d'application :** Web / SaaS
* **Protocole :** HTTPS (Port 443)
* **Accès :** Authentification HTTP Basic (Mire de connexion)

---

## 🛠️ Étapes de configuration

### Étape 1 : Ajout de l'URL dans la section HTTP

1. Dans Checkmk, allez dans le menu des **Services**.
2. Accédez à la section **HTTP** (*HTTP / HTTPS Services*).
3. Cliquez sur **Ajouter une règle / URL**.
4. Renseignez les paramètres de l'URL Ligeo :
   * **Nom du service :** `HTTP_Ligeo_Gestion`
   * **URL / Virtual Host :** `reference.gestion...ligeo-archives.com`
   * **SSL / TLS :** Activer l'option SSL sur le port **443** (HTTPS).
   
   > **Note sur l'authentification (Mire de connexion) :**
   > * **Option A (Recommandée) :** Dans la section *Authentication*, renseignez les identifiants HTTP Basic pour valider l'accès complet (Retour `200 OK`).
   > * **Option B :** Si aucun identifiant n'est renseigné, configurez le code de réponse attendu (*Expected HTTP response code*) sur `401` pour valider que le serveur répond.

---

### Étape 2 : Création de l'hôte spécifique à l'URL

1. Dans Checkmk, créez un nouvel hôte dédié à cette URL.
2. Renseignez les informations de l'hôte :
   * **Nom de l'hôte :** Indiquez un nom clair (ex: `URL-Ligeo-Gestion` ou le FQDN `reference.gestion...ligeo-archives.com`).
   * **Type d'hôte :** Sélectionnez **`Host without agent`** *(ou No IP / API integrations)*.

---

### Étape 3 : Activation et validation

1. Enregistrez les modifications (*Save*).
2. Validez les changements via le menu **Pending Changes** en cliquant sur **Activate on selected sites**.
3. Vérifiez dans la vue des services que l'hôte et son contrôle HTTP remontent bien au statut **OK** (Vert).
