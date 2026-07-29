# Procédure : Supervision de l'URL Ligeo Gestion dans Checkmk

Cette procédure détaille la méthode pour créer l'hôte et configurer la supervision HTTPS de l'application **Ligeo Gestion** dans Checkmk.

---

## 📌 Informations sur le service

* **Application :** Ligeo (Module Gestion)
* **Éditeur :** Boscop
* **Type d'hôte :** Application Web / URL
* **Protocole :** HTTPS (Port 443)
* **Accès :** Authentification HTTP Basic (Mire de connexion)

---

## 🛠️ Étapes de configuration

### Étape 1 : Création de l'hôte dédié à l'URL

1. Dans Checkmk, allez dans **Setup** > **Hosts** > **Add host** (dans le dossier `Surveillance par url http`).
2. Renseignez uniquement le nom :
   * **Nom de l'hôte :** `URL-Ligeo-Gestion`
3. Les paramètres réseau et étiquettes sont hérités automatiquement du dossier parent (`No IP`, `host:NoPing`).
4. Cliquez sur **Save & exit** (ou *Save & go to service configuration*).

---

### Étape 2 : Ajout de la règle de contrôle HTTP

1. Allez dans **Setup** > **Services** > **HTTP, TCP, Email, ...**
2. Dans la section **Networking**, cliquez sur **Check HTTP web service**.
3. Cliquez sur **Add rule** (ou *Create rule in folder*).
4. Configurez la règle :
   * **Service Name :** `HTTP Ligeo-Gestion`
   * **Name of virtual host / URL :** `reference.gestion...ligeo-archives.com`
   * **SSL/TLS :** Activer l'option SSL sur le port **443** (HTTPS).
   
   > **Gestion de la mire de connexion (401) :**
   > * **Option A (Authentifiée) :** Dans la section *Authentication*, saisissez des identifiants HTTP Basic valides (retour `200 OK`).
   > * **Option B (Disponibilité seule) :** Laissez vide et définissez le code de réponse attendu (*Expected HTTP response code*) sur `401`.

5. Dans la section **Conditions** > **Explicit hosts**, sélectionnez l'hôte créé à l'étape 1 (`URL-Ligeo-Gestion`).

---

### Étape 3 : Validation et Déploiement

1. Enregistrez la règle (**Save**).
2. Cliquez sur l'icône **Pending Changes** (en haut à droite) puis sur **Activate on selected sites**.
3. Vérifiez dans **Monitoring** que le service `HTTP Ligeo-Gestion` associé à l'hôte `URL-Ligeo-Gestion` est bien au statut **OK** (Vert).
