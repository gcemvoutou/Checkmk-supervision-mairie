# PROCÉDURE — Déploiement Wallboard CheckMK sur Dell OptiPlex 3020M (Ubuntu Desktop 24.04 LTS)

> Projet : Bureau du Desk — Mairie de Saint-Égrève
> Auteur : Clara — Apprentie BTS SIO SISR
> Matériel : Dell OptiPlex 3020M, Intel Core i3
> ⚠️ Version Desktop demandée par le responsable (pas Server, pour rester distinct du serveur CheckMK existant déjà en Ubuntu Server)

## Mon conseil sur l'approche kiosque

Vu qu'on part sur Desktop (donc GNOME déjà installé, contrairement à l'approche Server + Xorg/Openbox minimal qu'on avait prévue avant), autant **garder GNOME tel quel et configurer le kiosque par-dessus** plutôt que de se battre pour l'alléger :
- Retirer GNOME reviendrait presque à réinstaller un Server, ce qui contredit la demande du responsable
- GNOME sur un i3 avec au moins 4 Go de RAM tourne très bien pour un usage kiosque simple (un seul Chromium plein écran, pas de multitâche)
- L'autologin GDM + la désactivation des économiseurs d'écran/veille + un lancement automatique de Chromium au démarrage de session suffisent largement à obtenir le même résultat "panneau de gare" qu'avec Openbox, avec moins de configuration bas niveau à gérer

## Sommaire
1. [Vue d'ensemble & prérequis](#1-vue-densemble--prérequis)
2. [Vérifications matérielles sur place](#2-vérifications-matérielles-sur-place)
3. [Création de la clé USB bootable](#3-création-de-la-clé-usb-bootable)
4. [Remise à zéro du SSD avant installation](#4-remise-à-zéro-du-ssd-avant-installation)
5. [Installation d'Ubuntu Desktop 24.04 LTS](#5-installation-dubuntu-desktop-2404-lts)
6. [Réseau — coordination avec l'admin réseau](#6-réseau--coordination-avec-ladmin-réseau)
7. [Import du certificat WatchGuard (rootCA)](#7-import-du-certificat-watchguard-rootca)
8. [Installation de Chromium](#8-installation-de-chromium)
9. [Configuration du kiosque sur GNOME](#9-configuration-du-kiosque-sur-gnome)
10. [Résilience : anti-veille, watchdog, démarrage auto](#10-résilience--anti-veille-watchdog-démarrage-auto)
11. [Compte CheckMK dédié "kiosque"](#11-compte-checkmk-dédié-kiosque)
12. [Sécurité (SSH / pare-feu)](#12-sécurité-ssh--pare-feu)
13. [Tests de recette](#13-tests-de-recette)
14. [Suggestions / ajouts](#14-suggestions--ajouts)

---

## 1. Vue d'ensemble & prérequis

**Matériel :**
- [ ] Dell OptiPlex 3020M (Intel Core i3), SSD propre
- [ ] Câble HDMI adapté
- [ ] Clé USB ≥ 8 Go (Desktop est plus lourd que Server, prévois large)
- [ ] Câble Ethernet
- [ ] Clavier/souris temporaire pour l'installation

**À récupérer avant de te déplacer sur site :**
- [ ] ISO **Ubuntu Desktop 24.04 LTS** (et non plus Server) — https://ubuntu.com/download/desktop
- [ ] URL du dashboard : `http://192.168.198.21/monitoring`
- [ ] Le fichier **`certificate.crt`** trouvé sur le partage réseau (`K:\03_Systemes_Information\...`) — copié sur une clé USB
- [ ] Confirmation avec Zakk que c'est bien le bon rootCA (message rapide, cf. section 6)

---

## 2. Vérifications matérielles sur place

Avant même de lancer l'installation, vérifie les specs réelles de la machine — ça conditionne si GNOME tournera confortablement ou si on devra alléger certains réglages visuels.

Si tu peux déjà booter sur un live USB Ubuntu ou dans un BIOS qui affiche les specs :
- **RAM** : idéalement 4 Go minimum pour GNOME + Chromium en kiosque confortablement ; en dessous, ça peut ramer un peu mais reste jouable pour un usage à un seul onglet fixe
- **Stockage** : le SSD fourni doit être détecté correctement (vérifie dans le BIOS qu'il apparaît bien)

Une fois l'OS installé, tu pourras confirmer avec :
```bash
free -h
df -h
```

> 💡 Si jamais la RAM s'avère limite (2 Go ou moins), dis-le-moi, on ajoutera une désactivation de certains effets visuels GNOME (`gsettings set org.gnome.desktop.interface enable-animations false`) pour alléger la charge.

---

## 3. Création de la clé USB bootable

Identique à ce qu'on a déjà fait, juste avec l'ISO **Desktop** cette fois (plus volumineuse, prévois une clé de 8 Go minimum) :

**Linux (dd) :**
```bash
lsblk
sudo umount /dev/sdX*
sudo dd if=ubuntu-24.04-desktop-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

**Windows (Rufus) :**
1. Sélectionne la clé USB
2. Sélectionne l'ISO Ubuntu **Desktop** 24.04
3. Schéma de partition : GPT, Système de destination : UEFI (non CSM)
4. Démarrer

> 💡 Pense aussi à copier le fichier `certificate.crt` sur cette même clé USB (dans un dossier à part, ex. `certif/`), ça t'évite d'avoir besoin d'une deuxième clé sur place.

---

## 4. Remise à zéro du SSD avant installation

Le cahier des charges précise que le SSD doit arriver "effacé et propre" (fourni par ton collègue) — mais autant vérifier/forcer un vrai nettoyage toi-même avant l'installation, surtout si le disque a potentiellement déjà servi (ancien poste recyclé, test précédent, etc.). Deux niveaux possibles selon ton besoin :

### Option A — Suffisant dans la grande majorité des cas : laisser l'installeur Ubuntu tout gérer

L'installeur Ubuntu Desktop propose une option **"Effacer le disque et installer Ubuntu"** (vue en section 5, étape 6) qui reformate entièrement le disque et recrée les partitions proprement. Pour un disque "propre" comme annoncé, **c'est largement suffisant**, pas besoin d'étape manuelle en plus.

### Option B — Si tu veux repartir d'un disque vraiment "à blanc" (ex. données résiduelles d'un ancien usage, ou simplement par principe de prudence avant un déploiement en prod)

1. Démarre sur la clé USB Ubuntu, choisis **"Essayer Ubuntu"** (mode live, sans installer) pour avoir un environnement de travail
2. Ouvre un terminal (`Ctrl+Alt+T`)
3. Identifie le disque à effacer :
```bash
lsblk
```
Repère le bon disque (ex. `/dev/sda` ou `/dev/nvme0n1` selon le type de SSD) — ⚠️ vérifie bien la taille affichée pour ne pas te tromper de disque.

4. **Pour un SSD SATA classique**, un effacement rapide des tables de partitions suffit (pas besoin d'écraser tout le disque bit par bit comme sur un vieux HDD — sur SSD ça userait le disque pour rien) :
```bash
sudo wipefs -a /dev/sda
```

5. **Alternative plus poussée : secure erase natif du SSD** (efface électriquement toutes les cellules mémoire, la méthode la plus propre spécifiquement pour un SSD) :
```bash
sudo apt install -y hdparm
sudo hdparm --user-master u --security-set-pass p /dev/sda
sudo hdparm --user-master u --security-erase p /dev/sda
```
⚠️ Cette commande peut échouer si le SSD a un "security freeze" activé au boot (fréquent) — dans ce cas il faut parfois passer par une mise en veille/réveil du disque avant de relancer la commande. Pas obligatoire pour ton usage, l'option A ou le `wipefs` suffisent largement pour un wallboard.

6. Redémarre ensuite normalement sur la clé USB pour lancer la vraie installation (section 5).

> ⚠️ **Cas de ce PC précis (Dell OptiPlex 3020M déjà utilisé, état du SSD inconnu) : pars sur l'option B.** Comme la machine a servi avant et que tu ne sais pas ce qu'il reste dessus (données du précédent utilisateur), un vrai effacement depuis l'environnement live est plus sûr qu'un simple reformatage — d'autant plus pour un poste municipal où la confidentialité des anciennes données compte.
>
> ⚠️ **Important : impossible de faire cet effacement depuis PowerShell/Windows si Windows est encore installé et démarré sur ce disque.** Tu ne peux pas effacer le disque système sur lequel l'OS est actuellement en cours d'exécution — `diskpart clean` ou `Clear-Disk` refuseront ou échoueront. Il faut obligatoirement démarrer sur la clé USB Ubuntu en mode **"Essayer Ubuntu"** (live, sans installer), et effectuer l'effacement (étapes 2 à 5 ci-dessus) depuis cet environnement live, où le disque interne n'est plus le système actif et peut donc être manipulé librement.

---

## 5. Installation d'Ubuntu Desktop 24.04 LTS

1. Boot sur la clé USB (touche de boot selon le modèle Dell — souvent `F12` sur les OptiPlex)
2. Choisis **"Installer Ubuntu"** (pas "Essayer Ubuntu", même si tu peux passer par le mode Essai pour vérifier que tout est détecté avant de lancer l'install pour de vrai — pratique sur du matériel que tu ne connais pas encore)
3. Disposition du clavier : French
4. Type d'installation : **Installation normale** (pas "minimale", ici on garde GNOME complet volontairement)
5. Décoche "Télécharger les mises à jour pendant l'installation" si tu veux gagner du temps (tu feras un `apt update` après), ou coche-la si le réseau est bon ce jour-là
6. Effacer le disque et installer Ubuntu (le SSD est propre, pas de dual-boot à gérer)
7. Fuseau horaire : Europe/Paris
8. Compte utilisateur : nom `checkmk-kiosk`, coche **"Se connecter automatiquement"** ✔️ (ça configurera l'autologin GDM directement à l'installation, tu n'auras pas à le faire manuellement après)
9. Laisse l'installation se terminer, retire la clé USB, redémarre

✔️ **Vérification** : au redémarrage, la session doit s'ouvrir automatiquement sur le bureau GNOME sans demander de mot de passe.

---

## 6. Réseau — coordination avec l'admin réseau

### 6.1 — Récupérer la MAC réelle
```bash
ip link show
```
(Ouvre un terminal via le raccourci GNOME `Ctrl+Alt+T` ou depuis les applications)

### 6.2 — Message à Zakk (réservation DHCP)

> Objet : Wallboard bureau du desk — réservation DHCP
>
> Salut Zakk,
>
> Réservation DHCP fixe pour la machine `wallboard-desk` (Dell OptiPlex 3020M) :
> - Adresse MAC : `xx:xx:xx:xx:xx:xx`
> - Emplacement : bureau du desk
> - Usage : accès HTTP vers `192.168.198.21` (CheckMK) + SSH entrant pour maintenance depuis le VLAN IT
>
> Merci !

> 💡 **Tu n'as pas besoin d'attendre la confirmation de Zakk pour avancer.** Laisse la machine en DHCP dynamique classique pendant toute la phase de configuration (sections 7 à 12 : certificat, Chromium, kiosque GNOME, résilience). Rien de tout ça ne dépend d'une IP fixe. Ne bascule sur l'IP réservée/fixe (6.3 ci-dessous) qu'en toute fin de projet, une fois tout le reste validé et juste avant la recette finale — c'est à ce moment-là que ça devient utile, pour que l'admin réseau retrouve toujours la même adresse dans ses logs/pare-feu une fois la machine posée définitivement.

### 6.3 — IP fixe (à faire en toute fin de projet, une fois la réservation DHCP confirmée ; sinon config manuelle via NetworkManager)

Sur Desktop, la config réseau passe par **NetworkManager** plutôt que netplan brut :
```bash
nmcli connection show
nmcli connection modify "Wired connection 1" ipv4.addresses 192.168.198.XX/24 ipv4.gateway 192.168.198.1 ipv4.dns 192.168.198.1 ipv4.method manual
nmcli connection up "Wired connection 1"
```

✔️ Test :
```bash
ping 192.168.198.21
curl -I http://192.168.198.21/monitoring
```

---

## 7. Import du certificat WatchGuard (rootCA)

### 7.1 — Confirme avec Zakk que c'est le bon fichier

Message court avant de te déplacer :
> "J'ai trouvé un fichier `certificate.crt` (délivré à/par : rootCA) sur K:\03_Systemes_Information\... C'est bien le certificat racine du WatchGuard qu'il faut que j'installe sur un poste dédié pour qu'il puisse joindre le Snap Store en HTTPS ?"

### 7.2 — Copier le certificat depuis la clé USB

```bash
# Le point de montage exact peut varier, vérifie avec :
lsblk
# Généralement automonté sous /media/checkmk-kiosk/NOM_CLE/

sudo cp /media/checkmk-kiosk/*/certif/certificate.crt /usr/local/share/ca-certificates/rootca.crt
sudo update-ca-certificates
```
Tu dois voir `1 added` dans le message de sortie.

### 7.3 — Vérification

```bash
curl -v https://api.snapcraft.io 2>&1 | grep -i issuer
```
✔️ L'erreur `x509: certificate signed by unknown authority` doit avoir disparu.

---

## 8. Installation de Chromium

```bash
sudo snap install chromium
which chromium
```
Chemin attendu : `/snap/bin/chromium`.

---

## 9. Configuration du kiosque sur GNOME

### 9.1 — Désactiver l'écran de verrouillage et la mise en veille

```bash
gsettings set org.gnome.desktop.session idle-delay 0
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.desktop.screensaver idle-activation-enabled false
```

### 9.2 — Désactiver les notifications système (évite les popups qui viendraient polluer l'affichage)

```bash
gsettings set org.gnome.desktop.notifications show-banners false
```

### 9.3 — Masquer le curseur de souris en kiosque

```bash
sudo apt install -y unclutter
```
On l'ajoutera au lancement automatique juste après.

### 9.4 — Créer le script de lancement Chromium

```bash
sudo nano /usr/bin/chromium-wallboard.sh
```
```bash
#!/bin/bash
URL="http://192.168.198.21/monitoring"

unclutter -idle 0.1 -root &

chromium \
  --kiosk \
  --noerrdialogs \
  --disable-infobars \
  --disable-session-crashed-bubble \
  --disable-translate \
  --no-first-run \
  --start-fullscreen \
  --overscroll-history-navigation=0 \
  --check-for-update-interval=31536000 \
  "$URL"
```
```bash
sudo chmod +x /usr/bin/chromium-wallboard.sh
```

### 9.5 — Lancer ce script automatiquement à l'ouverture de session GNOME

```bash
mkdir -p ~/.config/autostart
nano ~/.config/autostart/wallboard.desktop
```
```ini
[Desktop Entry]
Type=Application
Exec=/usr/bin/chromium-wallboard.sh
Hidden=false
X-GNOME-Autostart-enabled=true
Name=Wallboard CheckMK
Comment=Lance le kiosque CheckMK au démarrage de session
```

✔️ **Test** : redémarre la machine. La session GNOME doit s'ouvrir automatiquement (autologin), puis Chromium doit se lancer en plein écran quelques secondes après.

---

## 10. Résilience : anti-veille, watchdog, démarrage auto

### 10.1 — Watchdog pour relancer Chromium en cas de crash

```bash
sudo nano /usr/bin/watchdog-wallboard.sh
```
```bash
#!/bin/bash
while true; do
  if ! pgrep -x "chromium" > /dev/null; then
    /usr/bin/chromium-wallboard.sh &
  fi
  sleep 15
done
```
```bash
sudo chmod +x /usr/bin/watchdog-wallboard.sh
```
Ajoute un second fichier autostart pour ce watchdog :
```bash
nano ~/.config/autostart/watchdog.desktop
```
```ini
[Desktop Entry]
Type=Application
Exec=/usr/bin/watchdog-wallboard.sh
Hidden=false
X-GNOME-Autostart-enabled=true
Name=Wallboard Watchdog
```

### 10.2 — Reprise après coupure de courant

Vérifie dans le BIOS Dell (touche `F2` au démarrage généralement sur les OptiPlex) l'option **"AC Recovery"** ou **"Power On After Power Failure"** → mets-la sur **"Power On"** (parfois appelée "On" tout court selon le BIOS Dell).

### 10.3 — Mises à jour automatiques sans reboot intempestif en journée

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```
Puis configure une fenêtre de reboot nocturne dans `/etc/apt/apt.conf.d/50unattended-upgrades` :
```
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

---

## 11. Compte CheckMK dédié "kiosque"

1. **Setup > Users** → Ajouter
2. Nom : `kiosque`
3. Mot de passe fort (gestionnaire de mots de passe pro)
4. Rôle : **guest/monitoring uniquement**, lecture seule
5. Pointe vers une vue/dashboard précis si possible

---

## 12. Sécurité (SSH / pare-feu)

```bash
sudo apt install -y openssh-server
sudo systemctl enable ssh
```
```bash
sudo nano /etc/ssh/sshd_config
```
```
PasswordAuthentication no
```
- SSH limité à une plage IP/VLAN admin, à valider avec Zakk
- Pense à désactiver le pare-feu GNOME/ufw seulement si besoin, ou configure une règle explicite pour autoriser le port 22 depuis le VLAN IT :
```bash
sudo ufw allow from 192.168.198.0/24 to any port 22
sudo ufw enable
```

---

## 13. Tests de recette

- [ ] PC installé et relié à la TV du desk
- [ ] Démarrage → session GNOME auto + Chromium plein écran sur CheckMK, sans intervention
- [ ] Certificat WatchGuard fonctionnel (`sudo snap refresh` sans erreur TLS)
- [ ] Test coupure d'alimentation réelle → reprise automatique complète
- [ ] Aucune veille d'écran ni verrouillage de session après plusieurs heures
- [ ] IP réservée stable après plusieurs redémarrages
- [ ] Accès SSH fonctionnel depuis un poste IT
- [ ] Curseur invisible en continu
- [ ] Crash volontaire Chromium (`pkill chromium`) → relance automatique en moins de 15s
- [ ] Aucune notification système ni popup ne vient perturber l'affichage

---

## 14. Suggestions / ajouts

- **RAM à vérifier en premier sur place** (`free -h`) — si elle s'avère juste (2 Go ou moins), prévoir de désactiver les animations GNOME (`gsettings set org.gnome.desktop.interface enable-animations false`) pour fluidifier.
- **Désactiver le économiseur d'écran GNOME au niveau du gestionnaire de connexion (GDM)** en plus de la session utilisateur, pour être sûre qu'aucune veille ne s'active même avant l'ouverture de session (utile si jamais l'autologin échoue un jour) :
  ```bash
  sudo nano /etc/gdm3/greeter.dconf-defaults
  ```
  ```
  [org/gnome/desktop/screensaver]
  idle-activation-enabled=false
  ```
- **Documenter dans ton GitHub la différence d'approche Server vs Desktop** pour ce projet précis — ça montre que tu sais adapter une solution à une contrainte organisationnelle (ici, la volonté du responsable de différencier ce poste du serveur CheckMK existant), un bon point à valoriser dans ton portfolio.
- **Wallboard natif CheckMK** : toujours une piste à explorer une fois le kiosque de base validé.
- **Anticipation d'une évolution vers plusieurs pages/vues affichées :** pas besoin de changer quoi que ce soit maintenant. Le script `chromium-wallboard.sh` pointe vers une seule URL en dur, c'est volontairement simple pour cette V1. Le jour où tu voudras afficher plusieurs dashboards en rotation, la solution la plus propre sera de basculer vers le **wallboard natif CheckMK** évoqué juste au-dessus — CheckMK gère nativement la rotation entre plusieurs vues avec son propre système de rafraîchissement. Tu n'auras alors qu'à changer l'URL cible dans le script (une seule ligne) pour pointer vers l'URL du wallboard au lieu d'une vue unique, plutôt que de repartir sur une architecture à onglets multiples/rotation custom côté Chromium, plus fragile à maintenir.
- **Étiquette physique** sur le boîtier avec hostname/IP/contact IT.

---

*Document rédigé dans le cadre du BTS SIO SISR — Mairie de Saint-Égrève*
*v3 — adaptée à Ubuntu Desktop 24.04 LTS sur Dell OptiPlex 3020M, intègre le retour d'expérience des tests VM (certificat WatchGuard)*
