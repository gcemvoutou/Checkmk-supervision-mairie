# 📁 Fiche Projet Portfolio — Supervision Réseau avec Checkmk

> **Administration et Optimisation de la Supervision Réseau**
> Direction des Systèmes d'Information (DSI) — Mairie (collectivité territoriale)

---

## 📋 Informations générales

| Champ | Détail |
|---|---|
| **Structure** | DSI — Mairie (collectivité territoriale) |
| **Rôle** | Technicien Système et Réseau (Alternant — BTS SIO SISR) |
| **Outil** | Checkmk Raw Edition (CRE) |
| **Architecture** | Site unique centralisé — serveur `monitoring` |
| **Périmètre** | 204 hôtes supervisés (parc en évolution continue), multi-sites municipaux |

---

## 1. Présentation du projet

La mairie gère une infrastructure informatique hétérogène répartie sur de nombreux sites distants (Hôtel de Ville, écoles, gymnases, services techniques).

L'objectif de ce projet est de **centraliser, maintenir à jour et piloter la supervision** de l'ensemble du parc afin de :

- Garantir la continuité du service public
- Anticiper les pannes matérielles ou réseau
- Fiabiliser l'inventaire des équipements supervisés

L'outil retenu est **Checkmk Raw Edition (CRE)**, déployé sur un serveur Linux local nommé `monitoring`. À ce jour, **204 hôtes** sont enregistrés dans la solution, un chiffre amené à évoluer au fil des mises en production et des retraits de matériel.

> [!NOTE]
> Checkmk Raw Edition (CRE) est la version open source gratuite de Checkmk. Elle offre toutes les fonctionnalités essentielles de supervision sans licence commerciale, ce qui la rend adaptée au contexte d'une collectivité territoriale.

<!-- 📸 CAPTURE SUGGÉRÉE : Vue d'ensemble du dashboard Checkmk (page d'accueil avec le résumé des hôtes OK / WARNING / CRITICAL) -->
![Dashboard Checkmk - Vue générale](./screenshots/01_dashboard_overview.png)

---

## 2. Compétences et missions réalisées

### 2.1 — Reprise et assainissement de l'existant

À la prise en charge du projet, la base de supervision contenait de nombreuses entrées obsolètes : équipements retirés du parc toujours présents dans Checkmk, adresses IP incorrectes générant des alertes erronées.

Un audit a été mené pour fiabiliser cet inventaire :

- Déplacements sur site pour vérifier physiquement si les équipements signalés **DOWN** étaient encore installés
- Correction des adresses IP obsolètes pour que les hôtes remontent correctement
- Intégration de **42 nouvelles bornes Wi-Fi** acquises par la mairie, configurées en supervision par ping
- Suppression des hôtes fantômes pour réduire le bruit dans les alertes

> [!IMPORTANT]
> Cette phase d'audit terrain est essentielle : un superviseur réseau n'a de valeur que si sa base de données reflète fidèlement la réalité de l'infrastructure. Des hôtes mal référencés génèrent des faux positifs et nuisent à la réactivité de l'équipe.

<!-- 📸 CAPTURE SUGGÉRÉE : Liste des hôtes dans Checkmk avec filtrage par état (ex : vue "All hosts" ou liste avec états DOWN / UNREACH) -->
![Liste des hôtes supervisés](./screenshots/02_hosts_list.png)

---

### 2.2 — Gestion du cycle de vie des hôtes

La base de supervision doit refléter en permanence la réalité du parc. Cela implique :

- **Communication** avec les équipes techniques de la DSI pour centraliser les demandes de modifications réseau
- **Ajout** de nouveaux hôtes lors de mises en production (nouveaux serveurs, nouvelles bornes)
- **Modification** des fiches hôtes lors de changements d'adressage IP
- **Suppression** des équipements retirés du parc pour maintenir un inventaire propre

> [!TIP]
> Dans Checkmk, toute modification de la configuration (ajout, suppression, modification d'un hôte) nécessite une étape d'**activation des changements** (`Activate pending changes`) pour être prise en compte. Penser à regrouper plusieurs modifications avant d'activer pour éviter des rechargements inutiles.

<!-- 📸 CAPTURE SUGGÉRÉE : Interface d'ajout d'un hôte dans Checkmk (formulaire "Add host") ou la vue "Pending changes" avant activation -->
![Ajout d'un hôte dans Checkmk](./screenshots/03_add_host.png)

---

### 2.3 — Implémentation des protocoles de supervision

Pour optimiser la collecte des données sans surcharger le réseau, la surveillance a été segmentée selon **4 méthodes**, adaptées à la criticité et à la nature de chaque équipement :

| Méthode | Équipements cibles | Indicateurs surveillés |
|---|---|---|
| **Agent Checkmk** | Serveurs Windows | Charge CPU, espace disque, RAM, état des services Windows critiques |
| **SNMP** | Switchs réseau, bornes Wi-Fi | État des ports, trafic (bande passante), Uptime |
| **URL / HTTP** | Chaufferies connectées, portail Intranet, portail de recrutement | Disponibilité (code HTTP 200), temps de réponse, validité du certificat SSL |
| **Ping (ICMP)** | Lecteurs de badges, bornes d'accès | Connectivité réseau de base, taux de perte de paquets, latence |

> [!NOTE]
> Le choix de la méthode de supervision dépend des capacités de l'équipement. Un lecteur de badges ne peut pas héberger un agent Checkmk ; le ping ICMP est alors la seule méthode viable pour vérifier sa présence sur le réseau.

<!-- 📸 CAPTURE SUGGÉRÉE : Fiche d'un hôte supervisé par agent (onglet "Services" avec CPU, disque, RAM visibles) -->
![Supervision par agent - Services d'un serveur Windows](./screenshots/04_agent_services.png)

<!-- 📸 CAPTURE SUGGÉRÉE : Fiche d'un hôte supervisé par SNMP (switch ou borne Wi-Fi avec état des interfaces) -->
![Supervision SNMP - État des ports d'un switch](./screenshots/05_snmp_switch.png)

---

### 2.4 — Exploitation et Maintien en Condition Opérationnelle (MCO)

- **Pilotage visuel** : supervision continue via un écran dédié (dashboard Checkmk) installé au sein du centre de support (Helpdesk) pour une réactivité maximale de l'équipe face aux incidents
- **Gestion des alertes** : suivi des notifications automatiques par mail lors des changements d'état (`OK` → `WARNING` → `CRITICAL` → `DOWN`)
- **Analyse des alertes répétitives** : distinction entre vraies pannes et faux positifs pour éviter l'alerte fatigue

> [!TIP]
> Les états Checkmk à connaître :
> - 🟢 `OK` — L'équipement fonctionne normalement
> - 🟡 `WARNING` — Seuil d'alerte atteint (ex : disque à 80 %)
> - 🔴 `CRITICAL` — Seuil critique atteint (ex : disque à 95 %)
> - ⚫ `DOWN` / `UNREACH` — L'hôte ne répond plus

<!-- 📸 CAPTURE SUGGÉRÉE : Vue du dashboard avec des hôtes en WARNING ou CRITICAL (ou simulation) pour illustrer le pilotage en temps réel -->
![Dashboard Checkmk - Alertes en cours](./screenshots/06_dashboard_alerts.png)

---

## 3. Difficultés rencontrées et solutions apportées

### 3.1 — Faux positifs sur les bornes Wi-Fi en adressage dynamique

Certaines bornes Wi-Fi apparaissaient en état **DOWN** dans Checkmk sans qu'aucune panne réelle ne soit constatée.

**Cause identifiée** : ces équipements obtenaient leur adresse IP via **DHCP**. Tout changement d'adresse rendait l'hôte injoignable pour le superviseur, qui le signalait donc comme indisponible.

**Blocage** : l'attribution d'une IP statique n'était pas possible directement, ces bornes n'étant pas rattachées à un **VLAN de management dédié**.

**Solution appliquée** : le problème a été **remonté à l'administrateur systèmes et réseaux** pour qu'une solution pérenne soit étudiée (création d'un VLAN management ou mise en place de réservations DHCP).

> [!IMPORTANT]
> Ce cas illustre une bonne pratique d'escalade : plutôt que de contourner le problème (ex : désactiver l'alerte), la cause racine a été identifiée et remontée au bon interlocuteur. Les équipements de supervision doivent idéalement être sur un **VLAN de management dédié** avec des adresses IP fixes pour garantir la fiabilité des données.

<!-- 📸 CAPTURE SUGGÉRÉE : Hôte en état DOWN dans Checkmk avec l'historique des changements d'état (pour illustrer les faux positifs répétitifs) -->
![Historique d'état d'un hôte - faux positifs DHCP](./screenshots/07_host_down_history.png)

---

### 3.2 — Inventaire initial incomplet et non fiabilisé

De nombreux équipements présents dans Checkmk n'existaient plus physiquement sur le réseau. Des **déplacements terrain** ont été nécessaires pour vérifier l'état réel du parc avant de nettoyer la base de supervision.

> [!NOTE]
> Ce type de dérive est courant dans les environnements où la supervision n'est pas maintenue activement. C'est pourquoi la gestion du cycle de vie des hôtes (section 2.2) est une mission à part entière et non une tâche ponctuelle.

---

## 4. Impact pour la collectivité

- ✅ **Proactivité** : détection des pannes avant les remontées d'incidents par les agents (ex : serveurs de fichiers saturés, pannes de lecteurs de badges, défaillances de chaufferies)
- ✅ **Fiabilisation de la base** : cartographie de supervision alignée avec la réalité du terrain, réduction du bruit dans les alertes
- ✅ **Réactivité améliorée** : le dashboard visible en permanence au Helpdesk permet une prise en charge rapide des incidents réseau
- ✅ **Escalade structurée** : identification et remontée d'un problème d'architecture réseau (absence de VLAN management) à l'administrateur systèmes et réseaux

---

## 5. Limites et perspectives

| Limite | Perspective d'évolution |
|---|---|
| Bornes Wi-Fi en adressage dynamique (faux positifs) | Création d'un VLAN management ou réservations DHCP |
| Supervision uniquement temps réel | Mise en place de rapports périodiques automatisés (fonctionnalité native Checkmk) |
| Pas de supervision des équipements hors réseau local | Étude d'un déploiement distribué si de nouveaux sites sont raccordés |

---

## 6. Compétences mobilisées

### Compétences techniques

- Administration de Checkmk Raw Edition (CRE)
- Supervision réseau : Agent, SNMP, HTTP/URL, ICMP (ping)
- Gestion d'adressage IP, notions DHCP et VLAN
- Administration Linux (serveur `monitoring`)
- Diagnostic réseau terrain

### Compétences transversales

- Communication avec les équipes techniques de la DSI
- Audit et vérification terrain
- Escalade et remontée d'incidents structurée
- Rigueur dans la tenue d'un inventaire à jour

---

*Document rédigé dans le cadre du BTS SIO SISR — Alternance DSI Mairie*
