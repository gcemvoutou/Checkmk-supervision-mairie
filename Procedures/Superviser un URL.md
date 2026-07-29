# Procédure : Supervision de l'URL Ligeo Gestion dans Checkmk

Cette procédure détaille la création de l'hôte et la configuration de la règle `Check HTTP web service` pour l'application **Ligeo Gestion**, conformément aux standards du service informatique.

---

## 📌 Informations sur le service

* **Application :** Ligeo (Module Gestion)
* **Éditeur :** Boscop
* **Type d'hôte :** Application Web / URL
* **Protocole :** HTTPS (Port 443)

---

## 🛠️ Étapes de configuration

### Étape 1 : Création de l'hôte dédié à l'URL

1. Dans Checkmk, allez dans **Setup** > **Hosts** > **Add host** (dans le dossier `Surveillance par url http`).
2. Renseignez les informations de l'hôte :
   * **Nom de l'hôte :** `URL-Ligeo-Gestion` (ou le nom FQDN du site)
3. Les paramètres réseau (`No IP`) et les étiquettes (`host:NoPing`) sont automatiquement hérités du dossier parent.
4. Cliquez sur **Save & exit**.

---

### Étape 2 : Configuration de la règle `Check HTTP web service`

1. Allez dans **Setup** > **Services** > **HTTP, TCP, Email, ...**
2. Dans la section **Networking**, cliquez sur **Check HTTP web service**.
3. Cliquez sur **Add rule** (ou *Create rule in folder*).
4. Dans la section **Value** > **HTTP web service endpoints to monitor** :
   * **Prefix :** Laissez `Use "HTTP(S)" as service name prefix`
   * **Name (required) :** Indiquez le nom de domaine (ex: `reference.gestion...ligeo-archives.com`)
   * **URL :** Renseignez l'URL complète avec le protocole :  
     `https://reference.gestion...ligeo-archives.com`

5. Dans la section **Standard settings for all endpoints** :
   * Cochez **Status code**.
   * Dans **Expected**, renseignez **`200`** *(ou `401` si le site renvoie une mire d'authentification sans identifiants)*.

6. Dans la section **Conditions** (en bas de page) :
   * Cochez **Explicit hosts** et sélectionnez l'hôte créé à l'étape 1 (`URL-Ligeo-Gestion`).

---

### Étape 3 : Validation et Déploiement

1. Cliquez sur **Save** en haut à gauche.
2. Cliquez sur l'icône jaune **Pending Changes** (en haut à droite), puis sur **Activate on selected sites**.
3. Accédez à la vue de l'hôte pour vérifier que le service web remonte bien au statut **OK** (Vert).
