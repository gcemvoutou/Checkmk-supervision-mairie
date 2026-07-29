# Procédure : Supervison de l'application Ligeo Gestion dans Checkmk

Cette procédure décrit la méthode pour ajouter et superviser l'application web hébergée **Ligeo Gestion** au sein de **Checkmk**, en prenant en compte l'accès sécurisé via HTTPS et l'authentification HTTP Basic.

---

## 📌 Informations sur le service

* **Application :** Ligeo (Module Gestion)
* **Éditeur :** Boscop
* **Type d'architecture :** SaaS / Web (client léger)
* **Port protocole :** HTTPS (443)
* **Méthode d'accès :** Authentification HTTP Basic

---

## 🛠️ Étapes de configuration dans Checkmk

### Étape 1 : Création / Déclaration de l'Hôte (Host)
1. Dans l'interface Checkmk, rendez-vous dans **Setup** > **Hosts** > **Add host**.
2. Renseignez les informations de l'hôte :
   * **Host name :** Indiquez le nom de domaine complet (ex: `reference.gestion...ligeo-archives.com`).
   * **IPv4 Address / DNS :** Renseignez le nom FQDN du serveur.
3. Dans la section **Monitoring agents**, sélectionnez :
   * `No IP / API integrations` (ou pas d'agent Checkmk local si l'hôte est supervisé uniquement à distance via HTTP).

---

### Étape 2 : Configuration du contrôle HTTP (Check HTTP service)

1. Allez dans **Setup** > **Services** > **HTTP / HTTPS Services** > **Check HTTP service**.
2. Créez une nouvelle règle (*Create rule in folder...*) et paramétrez les éléments suivants :

#### 🔹 Options du Service HTTP
* **Service Name :** `HTTP Ligeo-Gestion` *(ou tout autre nom explicite)*
* **Name of virtual host :** Saisissez le domaine complet (ex: `reference.gestion...ligeo-archives.com`).
* **Use SSL/TLS for the connection :** `Use SSL with port 443` *(Cocher la vérification du certificat SSL si nécessaire)*.

#### 🔹 Gestion de la mire d'authentification (Pop-up 401)
*Deux options sont possibles selon votre besoin :*

* **Option A : Test de l'accès sécurisé complet (Recommandé)**
  * Activez l'option **HTTP authentication (Basic Auth)**.
  * Renseignez le **Nom d'utilisateur** et le **Mot de passe** autorisés.
  * *Résultat attendu :* Code HTTP `200 OK`.

* **Option B : Test de la disponibilité globale du service (sans identifiants)**
  * Ne renseignez aucun identifiant.
  * Dans l'option **Expected HTTP response code**, indiquez : `401` (ou `200,401`).
  * *Résultat attendu :* Le service est considéré en **OK** dès lors que le serveur répond avec la demande d'authentification (code 401).

---

## 🔍 Validation et Déploiement

1. Cliquez sur **Save**.
2. Allez dans **Changes** (l'icône jaune en haut à droite) et validez avec **Activate on selected sites**.
3. Vérifiez dans **Monitoring** > **All hosts** > *Nom de l'hôte Ligeo* que le service `HTTP Ligeo-Gestion` remonte bien au statut **OK** (Vert).
