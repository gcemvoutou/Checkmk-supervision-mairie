# PROCÉDURE — Déploiement Wallboard CheckMK sur le mini PC réel (v2)

> Projet : Bureau du Desk — Mairie de Saint-Égrève
> Auteur : Clara — Apprentie BTS SIO SISR
> Cette version intègre le retour d'expérience du test en VM : blocage TLS lié à l'inspection SSL du pare-feu WatchGuard sur le trafic HTTPS sortant (Snap Store notamment).

## Sommaire
1. [Vue d'ensemble & prérequis](#1-vue-densemble--prérequis)
2. [Création de la clé USB bootable](#2-création-de-la-clé-usb-bootable)
3. [Installation d'Ubuntu Server](#3-installation-dubuntu-server)
4. [Réseau — coordination avec l'admin réseau (IP + certificat WatchGuard)](#4-réseau--coordination-avec-ladmin-réseau-ip--certificat-watchguard)
5. [Import du certificat WatchGuard ⚠️ étape critique](#5-import-du-certificat-watchguard--étape-critique)
6. [Environnement graphique minimal (Xorg + Openbox)](#6-environnement-graphique-minimal-xorg--openbox)
7. [Installation et configuration de Chromium en kiosque](#7-installation-et-configuration-de-chromium-en-kiosque)
8. [Résilience : démarrage auto, watchdog, anti-veille](#8-résilience--démarrage-auto-watchdog-anti-veille)
9. [Compte CheckMK dédié "kiosque"](#9-compte-checkmk-dédié-kiosque)
10. [Sécurité (SSH / pare-feu)](#10-sécurité-ssh--pare-feu)
11. [Tests de recette](#11-tests-de-recette)
12. [Suggestions / ajouts](#12-suggestions--ajouts)

---

## 1. Vue d'ensemble & prérequis

**Matériel :**
- [ ] Mini PC fourni par le collègue, SSD propre
- [ ] Câble HDMI
- [ ] Clé USB ≥ 4 Go
- [ ] Câble Ethernet
- [ ] Clavier/souris temporaire pour l'installation

**À récupérer AVANT de te déplacer sur site, cette fois-ci :**
- [ ] ISO Ubuntu Server LTS 24.04
- [ ] URL du dashboard : `http://192.168.198.21/monitoring`
- [ ] ⚠️ **Le fichier de certificat racine WatchGuard** (`.crt` ou `.pem`) — demande-le à Zakk en amont, c'est le point bloquant qu'on a identifié en test VM (voir section 5)
- [ ] Confirmation de la réservation DHCP (MAC à transmettre une fois sur le vrai matériel)

> ⚠️ **Ne pars pas sur site sans le certificat WatchGuard.** Sans lui, l'installation de Chromium (qui passe par le Snap Store) va bloquer exactement comme en test VM, et tu n'auras personne sous la main pour te le fournir dans l'urgence si Zakk n'est pas disponible ce jour-là.

---

## 2. Création de la clé USB bootable

Identique au test VM — voir méthode `dd` (Linux) ou Rufus (Windows), déjà validée. Pas de changement ici.

```bash
lsblk
sudo umount /dev/sdX*
sudo dd if=ubuntu-24.04-live-server-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

---

## 3. Installation d'Ubuntu Server

Identique à la procédure de test, avec :
- Type d'installation : **Ubuntu Server minimized**
- Réseau : DHCP dans un premier temps (on branchera la réservation fixe une fois l'IP confirmée par l'admin, section 4)
- **Coche "Install OpenSSH server"** ✔️
- Nom de machine : `wallboard-desk`
- Utilisateur : `checkmk-kiosk`

---

## 4. Réseau — coordination avec l'admin réseau (IP + certificat WatchGuard)

### 4.1 — Récupérer l'adresse MAC réelle

```bash
ip link show
```

### 4.2 — Message groupé à envoyer à Zakk

Regroupe ces **deux demandes en un seul message**, pour ne pas multiplier les allers-retours :

> Objet : Wallboard bureau du desk — réservation DHCP + certificat WatchGuard
>
> Salut Zakk,
>
> Pour le wallboard CheckMK du bureau du desk, j'aurais besoin de deux choses :
>
> 1. **Réservation DHCP fixe** pour la machine `wallboard-desk` :
>    - Adresse MAC : `xx:xx:xx:xx:xx:xx`
>    - Emplacement : bureau du desk
>    - Usage : accès HTTP vers `192.168.198.21` (CheckMK) + SSH entrant pour maintenance depuis le VLAN IT
>
> 2. **Le certificat racine du pare-feu WatchGuard (Fireware HTTPS Proxy)** au format `.crt` ou `.pem`. La machine doit installer Chromium via Snap, qui passe par du HTTPS externe (Snap Store) — sans ce certificat, l'inspection SSL du WatchGuard bloque la connexion (erreur `x509: certificate signed by unknown authority`, testé et confirmé en environnement VM).
>
> Merci d'avance !

### 4.3 — Configuration IP fixe (netplan) une fois l'IP confirmée

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: no
      addresses:
        - 192.168.198.XX/24
      routes:
        - to: default
          via: 192.168.198.1
      nameservers:
        addresses: [192.168.198.1, 8.8.8.8]
```
```bash
sudo netplan apply
```

✔️ **Test :**
```bash
ping 192.168.198.21
curl -I http://192.168.198.21/monitoring
```

---

## 5. Import du certificat WatchGuard ⚠️ étape critique

Cette étape doit être faite **avant** toute tentative d'installation de Chromium/Snap. C'est la correction directe du blocage rencontré en test VM.

### 5.1 — Transférer le certificat sur la machine

Si tu as le fichier sur une clé USB ou via SCP depuis ton PC :
```bash
scp watchguard.crt checkmk-kiosk@192.168.198.XX:~/
```

### 5.2 — Installer le certificat dans le magasin système

```bash
sudo cp ~/watchguard.crt /usr/local/share/ca-certificates/watchguard.crt
sudo update-ca-certificates
```
Tu dois voir un message confirmant qu'un certificat a été ajouté (`1 added`).

### 5.3 — Redémarrer les services concernés

```bash
sudo systemctl restart snapd
```

### 5.4 — Vérification avant de continuer

```bash
curl -v https://api.snapcraft.io 2>&1 | grep -i "issuer"
```
✔️ Tu dois maintenant voir l'issuer réel de Canonical/Let's Encrypt et non plus "Fireware HTTPS Proxy" comme erreur bloquante — ou en tout cas, la commande ne doit plus renvoyer d'erreur de certificat.

> 💡 Si tu n'as **toujours pas** le certificat au moment de l'installation (Zakk indisponible, etc.), utilise en secours **Epiphany** (`sudo apt install epiphany-browser`) qui s'installe en `.deb` classique sans passer par HTTPS/Snap — ça permet de ne pas bloquer le déploiement, quitte à migrer vers Chromium plus tard une fois le certificat récupéré.

---

## 6. Environnement graphique minimal (Xorg + Openbox)

```bash
sudo apt update
sudo apt install --no-install-recommends xserver-xorg x11-xserver-utils xinit openbox unclutter
```

```bash
mkdir -p ~/.config/openbox
nano ~/.config/openbox/autostart
```
```bash
xset s off
xset s noblank
xset -dpms

unclutter -idle 0.1 -root &

/usr/bin/chromium-wallboard.sh &
```

---

## 7. Installation et configuration de Chromium en kiosque

**Maintenant que le certificat est en place (section 5), l'installation devrait passer normalement :**

```bash
sudo snap install chromium
which chromium
```

Tu dois obtenir un chemin type `/snap/bin/chromium`.

### Script de lancement (attention au nom de commande corrigé : `chromium`, pas `chromium-browser`)

```bash
sudo nano /usr/bin/chromium-wallboard.sh
```
```bash
#!/bin/bash
URL="http://192.168.198.21/monitoring"

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

### Autologin + startx (inchangé)

```bash
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d
sudo nano /etc/systemd/system/getty@tty1.service.d/override.conf
```
```ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin checkmk-kiosk --noclear %I $TERM
```
```bash
nano ~/.bash_profile
```
```bash
if [ -z "$DISPLAY" ] && [ "$(tty)" = "/dev/tty1" ]; then
  startx
fi
```
```bash
sudo systemctl daemon-reload
sudo reboot
```

✔️ **Vérification post-reboot** (en SSH depuis un autre poste) :
```bash
ps aux | grep -E "Xorg|openbox|chromium"
```
Tu dois voir les trois process actifs.

---

## 8. Résilience : démarrage auto, watchdog, anti-veille

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
Ajoute dans `~/.config/openbox/autostart` :
```bash
/usr/bin/watchdog-wallboard.sh &
```

### Reprise après coupure de courant

Vérifie dans le BIOS/UEFI du mini PC l'option **"Restore on AC Power Loss"** → **ON**. Point à valider avec le collègue qui fournit le matériel, propre à chaque modèle.

---

## 9. Compte CheckMK dédié "kiosque"

1. **Setup > Users** → Ajouter
2. Nom : `kiosque`
3. Mot de passe fort (gestionnaire de mots de passe pro)
4. Rôle : **guest/monitoring uniquement**, lecture seule
5. Pointe si possible directement vers une vue/dashboard précis plutôt que la page d'accueil générale

---

## 10. Sécurité (SSH / pare-feu)

```bash
sudo nano /etc/ssh/sshd_config
```
```
PasswordAuthentication no
```
- SSH limité à une plage IP/VLAN admin, à valider avec Zakk au moment des règles de pare-feu
- Mises à jour de sécurité automatiques configurées avec fenêtre de reboot nocturne (`unattended-upgrades`)

---

## 11. Tests de recette

- [ ] PC installé et relié à la TV du desk
- [ ] Démarrage → Chromium plein écran sur CheckMK, sans intervention
- [ ] Test coupure d'alimentation réelle (débrancher/rebrancher) → reprise automatique
- [ ] Aucune veille d'écran après plusieurs heures
- [ ] IP réservée stable après plusieurs redémarrages
- [ ] Accès SSH fonctionnel
- [ ] Curseur invisible en permanence
- [ ] Crash volontaire Chromium → relance automatique en moins de 15s
- [ ] Certificat WatchGuard fonctionnel : `sudo snap refresh` ne renvoie plus d'erreur TLS

---

## 12. Suggestions / ajouts

- **Documenter le souci de certificat dans ton dépôt GitHub** : c'est un vrai point technique intéressant à valoriser dans ton portfolio (compréhension de l'inspection SSL, résolution méthodique via SSH/curl/openssl) — ça montre une vraie démarche de diagnostic, pas juste du "copier-coller de commandes".
- **Vérifier si d'autres services auront besoin du même certificat** à l'avenir (mises à jour automatiques `unattended-upgrades` si configurées en HTTPS, tout autre outil ajouté plus tard) — le certificat une fois importé couvre tous les cas, mais bon réflexe de le garder en tête.
- **Wallboard natif CheckMK** : toujours d'actualité, à explorer si tu as le temps une fois le kiosque de base validé.
- **Étiquette physique + petit doc de maintenance** collée sur le boîtier (hostname, IP, contact IT) pour tes successeurs.

---

*Document rédigé dans le cadre du BTS SIO SISR — Mairie de Saint-Égrève*
*v2 — intègre le correctif du blocage TLS WatchGuard identifié lors du test en VM VirtualBox*
