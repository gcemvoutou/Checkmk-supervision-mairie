# PROCÉDURE DÉTAILLÉE — Déploiement Wallboard CheckMK sur Dell OptiPlex 3020M

> Écrite pour quelqu'un qui n'a jamais fait ça. Chaque commande est expliquée : ce qu'elle fait, pourquoi on la lance, et à quoi t'attendre comme résultat.
> Matériel : Dell OptiPlex 3020M, Intel Core i3, 7,9 Go de RAM
> OS : Ubuntu Desktop 24.04 LTS

## Petit lexique avant de commencer

Quelques mots que tu vas croiser tout du long, définis une bonne fois pour toutes :

- **Terminal** : une fenêtre où on tape des commandes en texte au lieu de cliquer avec la souris. Sur Ubuntu, tu l'ouvres avec le raccourci `Ctrl + Alt + T`.
- **`sudo`** : à mettre devant une commande pour dire "fais-le avec les droits d'administrateur". Le système va te demander ton mot de passe la première fois, puis plus pendant quelques minutes.
- **Kiosque (mode kiosque)** : un navigateur affiché en plein écran, sans barre d'adresse, sans onglets, sans rien d'autre — comme un panneau d'affichage figé sur une seule page.
- **DHCP** : le mode par défaut où c'est le réseau qui attribue automatiquement une adresse IP à ta machine quand elle démarre (comme quand ton téléphone se connecte au wifi sans que tu configures rien).
- **IP fixe / réservation DHCP** : au lieu de laisser le réseau donner une IP au hasard, on demande à ce que la machine ait toujours la même adresse. Utile pour que l'admin réseau la retrouve facilement dans ses outils.
- **SSH** : une façon de se connecter à distance à une machine, en tapant des commandes, sans être physiquement devant l'écran/clavier de cette machine.
- **Certificat / TLS / HTTPS** : quand un site est en HTTPS (le petit cadenas dans le navigateur), la connexion est chiffrée, et un "certificat" sert à prouver que le site est bien celui qu'il prétend être. Un pare-feu d'entreprise (comme le WatchGuard de la mairie) peut "intercepter" ces connexions pour les vérifier, ce qui demande d'installer un certificat spécial sur la machine pour qu'elle fasse confiance à cette interception.
- **Snap** : un format d'installation de logiciels sur Ubuntu (comme un `.exe` sur Windows, mais en plus confiné). Chromium s'installe via ce système.

---

## Sommaire
1. [Ce qu'il te faut avant de commencer](#1-ce-quil-te-faut-avant-de-commencer)
2. [Créer la clé USB bootable](#2-créer-la-clé-usb-bootable)
3. [Effacer le disque dur de l'ancien PC](#3-effacer-le-disque-dur-de-lancien-pc)
4. [Installer Ubuntu Desktop](#4-installer-ubuntu-desktop)
5. [Premiers pas : ouvrir un terminal et se repérer](#5-premiers-pas--ouvrir-un-terminal-et-se-repérer)
6. [Connecter la machine au réseau](#6-connecter-la-machine-au-réseau)
7. [Installer le certificat de l'entreprise](#7-installer-le-certificat-de-lentreprise)
8. [Installer le navigateur Chromium](#8-installer-le-navigateur-chromium)
9. [Configurer le mode kiosque](#9-configurer-le-mode-kiosque)
10. [Empêcher l'écran de se mettre en veille](#10-empêcher-lécran-de-se-mettre-en-veille)
11. [Rendre le système résistant aux pannes](#11-rendre-le-système-résistant-aux-pannes)
12. [Créer le compte CheckMK dédié](#12-créer-le-compte-checkmk-dédié)
13. [Sécuriser l'accès à distance](#13-sécuriser-laccès-à-distance)
14. [Vérifier que tout fonctionne (recette)](#14-vérifier-que-tout-fonctionne-recette)

---

## 1. Ce qu'il te faut avant de commencer

Fais cette liste avant de te déplacer, pour ne rien découvrir sur place :

- [ ] Une **clé USB vide** d'au moins 8 Go (elle va être totalement effacée, donc pas de données importantes dessus)
- [ ] Un fichier **image disque** (fichier `.iso`) d'Ubuntu Desktop 24.04 LTS, à télécharger sur https://ubuntu.com/download/desktop — c'est le fichier qui contient tout le système à installer
- [ ] Le fichier de **certificat** trouvé sur le partage réseau (`certificate.crt`) — copie-le aussi sur ta clé USB, dans un petit dossier séparé (ex. `certif/`), pour l'avoir sous la main
- [ ] Un câble HDMI et un câble Ethernet
- [ ] Un clavier et une souris (temporaires, juste pour l'installation — tu n'en auras plus besoin après)
- [ ] L'URL du dashboard à afficher : `http://192.168.198.21/monitoring`

---

## 2. Créer la clé USB bootable

**Ce qu'on fait ici et pourquoi :** on transforme la clé USB en disque de démarrage, pour que le PC puisse "booter" (démarrer) dessus au lieu de démarrer sur son disque dur habituel. C'est comme ça qu'on va pouvoir installer Ubuntu.

### Si tu fais ça depuis un PC Windows (le plus simple)

1. Télécharge le logiciel **Rufus** (gratuit, pas besoin de l'installer, juste double-cliquer dessus) : https://rufus.ie/
2. Branche ta clé USB
3. Ouvre Rufus. Dans le champ "Périphérique", vérifie qu'il a bien sélectionné ta clé USB (attention si tu as plusieurs clés/disques branchés, ne te trompe pas — ça va tout effacer)
4. Clique sur "Sélection" et choisis ton fichier `.iso` d'Ubuntu Desktop téléchargé
5. Laisse les autres réglages par défaut (Rufus les ajuste automatiquement selon l'ISO)
6. Clique sur le bouton **"DÉMARRER"**
7. Une fenêtre peut te demander de confirmer que tu veux effacer la clé — clique sur "OK" (vérifie une dernière fois que c'est bien la bonne clé)
8. Attends la fin (une barre de progression s'affiche), ça prend généralement 5 à 10 minutes

### Si tu fais ça depuis un PC Linux

Ouvre un terminal (`Ctrl + Alt + T`), puis :

```bash
lsblk
```
Cette commande affiche la liste de tous les disques connectés à ta machine. Repère ta clé USB dans la liste — regarde la **taille** affichée pour être sûre de ne pas te tromper avec ton disque dur principal. Elle apparaît en général sous un nom comme `sdb` ou `sdc`.

```bash
sudo dd if=chemin/vers/ubuntu-24.04-desktop-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```
Remplace `chemin/vers/ubuntu-24.04-desktop-amd64.iso` par l'emplacement réel de ton fichier téléchargé, et `/dev/sdX` par le nom exact de ta clé (vu avec `lsblk` juste avant — par exemple `/dev/sdb`, **sans** chiffre à la fin).

⚠️ Cette commande écrase tout le contenu du disque que tu indiques, sans demander de confirmation. Vérifie bien deux fois le nom avant d'appuyer sur Entrée.

Ça prend quelques minutes, et rien ne s'affiche pendant un moment — c'est normal, ça travaille en silence jusqu'à la fin.

---

## 3. Effacer le disque dur de l'ancien PC

**Pourquoi cette étape :** ce PC (Dell OptiPlex 3020M) a déjà été utilisé avant, et tu ne sais pas ce qu'il reste dessus comme données de l'ancien utilisateur. Avant d'installer un nouveau système, on va nettoyer complètement le disque.

**Point important : tu ne peux pas faire ça depuis Windows si Windows est le système actuellement démarré sur ce disque.** On ne peut pas effacer le sol sur lequel on est en train de marcher. Il faut d'abord démarrer sur la clé USB qu'on vient de préparer, dans un mode qui ne touche pas encore au disque, pour pouvoir l'effacer depuis là.

### 3.1 — Démarrer sur la clé USB

1. Éteins le PC s'il est allumé
2. Branche la clé USB
3. Allume le PC et appuie plusieurs fois sur la touche qui ouvre le menu de démarrage — sur les PC Dell, c'est généralement **F12** (appuie dès que le logo Dell apparaît)
4. Un petit menu s'affiche avec la liste des disques de démarrage disponibles — choisis ta clé USB dans la liste (elle apparaît généralement avec son nom de marque, ex. "SanDisk USB Device")
5. Le PC démarre alors sur Ubuntu depuis la clé, et affiche un écran avec deux gros boutons : **"Essayer Ubuntu"** et **"Installer Ubuntu"**
6. Clique sur **"Essayer Ubuntu"** (pas "Installer" pour l'instant) — ça lance un Ubuntu temporaire en mémoire, sans rien installer sur le disque dur, ce qui nous permet justement de le manipuler librement

### 3.2 — Ouvrir un terminal et identifier le disque

Une fois sur le bureau Ubuntu "d'essai", ouvre un terminal avec `Ctrl + Alt + T`.

Tape :
```bash
lsblk
```
Ça liste tous les disques. Cette fois, cherche le **disque interne** du PC (pas la clé USB sur laquelle tu as démarré) — regarde la taille pour l'identifier (le SSD interne, souvent nommé `sda`, sera d'une taille différente de ta clé USB).

### 3.3 — Effacer le disque

Cette commande "efface les tables de partitions" du disque — en clair, elle supprime toute trace de l'organisation précédente du disque (les partitions Windows éventuelles, les données, etc.), pour repartir sur une base neutre :

```bash
sudo wipefs -a /dev/sda
```
Remplace `/dev/sda` par le nom réel du disque interne que tu as identifié à l'étape précédente. `sudo` te demandera peut-être un mot de passe — sur ce mode "essai", il n'y en a généralement pas, appuie juste sur Entrée si demandé.

✔️ Une fois la commande terminée (ça prend quelques secondes), le disque est prêt à recevoir une toute nouvelle installation, sans rien de l'ancien système.

### 3.4 — Relancer l'installation

Redémarre le PC (menu en haut à droite du bureau → icône d'alimentation → Redémarrer), retire la clé USB pendant le redémarrage si le PC te le demande, puis remets-la et refais l'étape "démarrer sur la clé USB" — mais cette fois, choisis **"Installer Ubuntu"** au lieu de "Essayer Ubuntu".

---

## 4. Installer Ubuntu Desktop

Tu es maintenant dans l'installeur pour de vrai. Suis les écrans dans l'ordre :

1. **Disposition du clavier** : choisis "French" (clavier français, AZERTY)
2. **Type d'installation** : choisis **"Installation normale"** (celle qui installe le bureau complet avec les applications de base — c'est ce qu'on veut, pas la version "minimale")
3. Tu peux décocher "Télécharger les mises à jour pendant l'installation" pour aller plus vite (tu les feras après)
4. **Type d'installation du disque** : choisis **"Effacer le disque et installer Ubuntu"** — comme on a déjà nettoyé le disque à l'étape 3, ça va juste créer les nouvelles partitions proprement dessus
5. Une fenêtre de confirmation s'affiche, listant ce qui va être fait sur le disque — clique sur "Installer maintenant" puis "Continuer"
6. **Fuseau horaire** : Europe/Paris (généralement détecté automatiquement)
7. **Créer ton compte utilisateur** :
   - Ton nom : ce que tu veux (ex. "Wallboard Desk")
   - Nom de la machine (ordinateur) : `wallboard-desk`
   - Nom d'utilisateur : `checkmk-kiosk`
   - Mot de passe : choisis-en un solide, note-le bien quelque part de sûr
   - **Coche impérativement la case "Se connecter automatiquement"** ✔️ — c'est ce qui permettra à la machine de démarrer directement sur le bureau sans qu'on ait à taper de mot de passe à chaque redémarrage, indispensable pour un affichage autonome
8. Laisse l'installation se dérouler (ça prend en général 15 à 25 minutes selon la machine et la connexion réseau)
9. À la fin, un message te demande de retirer le support d'installation (ta clé USB) et d'appuyer sur Entrée pour redémarrer

✔️ **Vérification :** à ce redémarrage, le bureau Ubuntu doit s'afficher directement, sans écran de connexion demandant un mot de passe. Si un écran de connexion apparaît quand même, ce n'est pas grave, on réglera l'autologin plus tard (section 11) si besoin.

---

## 5. Premiers pas : ouvrir un terminal et se repérer

À partir de maintenant, presque toutes les manipulations se font dans un terminal. Voici comment t'y retrouver :

**Ouvrir un terminal :** raccourci clavier `Ctrl + Alt + T`, ou clique sur l'icône "Activités" en haut à gauche, tape "Terminal", puis clique dessus.

**Comment lire une commande dans ce document :** chaque bloc gris est à copier-coller (ou taper) tel quel dans le terminal, puis valider avec la touche Entrée. Par exemple :
```bash
ls
```
Cette commande liste les fichiers du dossier où tu te trouves. Pas besoin de la lancer maintenant, c'est juste un exemple.

**Le mot de passe ne s'affiche pas quand tu le tapes** (ni étoiles, ni points) — c'est normal, une sécurité de Linux. Tape-le à l'aveugle et valide avec Entrée.

**Copier-coller dans le terminal :** le raccourci classique `Ctrl+V` ne fonctionne pas toujours dans un terminal Linux. Utilise plutôt `Ctrl + Shift + V`, ou fais un clic droit → "Coller".

---

## 6. Connecter la machine au réseau

**Ce qu'on fait :** on branche la machine au réseau de la mairie, on relève ses informations réseau actuelles, et on les transmet à l'admin réseau pour qu'il configure une règle de pare-feu et une IP fixe adaptée.

> ⚠️ **Mise à jour importante :** ton poste (`10.x.x.x`) et le serveur CheckMK (`192.168.198.21`) sont sur deux plages IP différentes qui ne communiquent pas par défaut. L'admin réseau va créer une règle de pare-feu pour autoriser ce dialogue, **et passer ta machine en IP fixe** (pas juste une réservation DHCP cette fois, une vraie IP statique). Pour ça, il a besoin de connaître ta configuration réseau **actuelle** (celle attribuée automatiquement par le DHCP).

1. Branche le câble Ethernet entre le PC et une prise réseau murale (ou le switch)
2. Ubuntu détecte généralement automatiquement la connexion et attribue une adresse IP toute seule (en DHCP, la valeur par défaut pour l'instant)

### 6.1 — Relever l'IP, le masque et la passerelle actuels

Ouvre un terminal et tape :
```bash
ip a
```
Cherche la ligne qui commence par `inet`, suivie d'une adresse du type `10.X.X.X/24` (ou un autre chiffre après le `/`, c'est le masque exprimé en notation courte — note ce chiffre aussi, ex. `/24`).

Pour connaître la passerelle (la porte de sortie de ton réseau local, souvent la première adresse de la plage) :
```bash
ip route | grep default
```
Ça affiche une ligne du type `default via 10.X.X.1 dev enp0s3` — l'adresse après `via` est ta passerelle.

### 6.2 — Transmettre ces informations à l'admin réseau

Il t'a demandé : IP actuelle, masque, passerelle. Envoie-lui simplement ces trois valeurs relevées à l'étape précédente.

### 6.3 — Une fois qu'il te communique la nouvelle IP fixe

Il va te répondre avec la configuration à appliquer : une nouvelle adresse IP fixe (probablement toujours en `10.x.x.x`, ou peut-être une adresse dédiée sur une autre plage — suis simplement ce qu'il te donne), le masque, la passerelle, et éventuellement un serveur DNS.

Applique cette configuration avec la commande suivante (remplace les valeurs entre `<>` par celles qu'il t'a données) :
```bash
sudo nmcli connection modify "Wired connection 1" ipv4.addresses <NOUVELLE_IP>/<MASQUE> ipv4.gateway <PASSERELLE> ipv4.dns <DNS> ipv4.method manual
sudo nmcli connection up "Wired connection 1"
```

> 💡 Si le nom de la connexion n'est pas exactement `"Wired connection 1"` chez toi, regarde son nom réel avec :
> ```bash
> nmcli connection show
> ```
> et utilise le nom affiché à la place.

### 6.4 — Vérifier que tout fonctionne après le changement

```bash
ip a
```
Confirme que la nouvelle IP fixe est bien appliquée (elle doit apparaître directement, sans attendre de bail DHCP).

```bash
ping 192.168.198.21
curl -I http://192.168.198.21/monitoring
```
Une fois que l'admin a bien mis en place sa règle de pare-feu de son côté, ces deux commandes doivent maintenant réussir — c'est le vrai test de validation de tout ce chantier réseau.

---

## 7. Installer le certificat de l'entreprise

**Pourquoi cette étape :** le pare-feu de la mairie (un boîtier appelé WatchGuard) inspecte les connexions sécurisées (HTTPS) qui sortent vers internet. Pour que ça se passe, il se fait passer pour un intermédiaire de confiance — mais pour que la machine accepte ça sans le voir comme une intrusion, il faut lui donner le "certificat" du WatchGuard, qui prouve qu'il est légitime. Sans cette étape, l'installation de Chromium (section suivante) va échouer avec une erreur de sécurité.

### 7.1 — Brancher la clé USB contenant le certificat

Branche la clé USB sur laquelle tu avais copié le fichier `certificate.crt` (préparée en section 1).

### 7.2 — Trouver où la clé a été montée

Sur Ubuntu Desktop, une clé USB branchée s'ouvre généralement automatiquement dans une fenêtre. Pour connaître son emplacement exact en ligne de commande, tape :
```bash
lsblk
```
Repère ta clé dans la liste, tu verras à côté un chemin du type `/media/checkmk-kiosk/NOM_DE_LA_CLE`.

### 7.3 — Copier le certificat au bon endroit

Cette commande copie ton fichier certificat dans le dossier où le système va chercher les certificats de confiance supplémentaires :
```bash
sudo cp /media/checkmk-kiosk/NOM_DE_LA_CLE/certif/certificate.crt /usr/local/share/ca-certificates/rootca.crt
```
Remplace `NOM_DE_LA_CLE` par le nom exact que tu as vu à l'étape précédente.

Puis, cette commande dit au système de "recharger" sa liste de certificats de confiance en tenant compte du nouveau fichier ajouté :
```bash
sudo update-ca-certificates
```
Tu dois voir un message affichant quelque chose comme `1 added` — ça confirme que le certificat a bien été pris en compte.

### 7.4 — Vérifier que ça fonctionne

```bash
curl -v https://api.snapcraft.io 2>&1 | grep -i issuer
```
Cette commande teste une connexion sécurisée vers un serveur externe et affiche qui a "signé" le certificat rencontré. Si tu vois un nom cohérent avec ton certificat (et pas de message d'erreur mentionnant "unknown authority" / "autorité inconnue"), c'est validé.

---

## 8. Installer le navigateur Chromium

**Ce qu'on fait :** on installe le navigateur qui va afficher le dashboard CheckMK en plein écran.

```bash
sudo snap install chromium
```
Cette commande télécharge et installe Chromium depuis le "Snap Store" (une sorte de magasin d'applications Ubuntu). Ça peut prendre quelques minutes selon la vitesse du réseau (le téléchargement fait plusieurs centaines de mégaoctets).

**Pour vérifier que c'est bien installé :**
```bash
which chromium
```
Tu dois voir s'afficher un chemin comme `/snap/bin/chromium` — ça confirme que la commande `chromium` existe bien sur ce système et qu'on pourra l'utiliser dans le script de kiosque juste après.

---

## 9. Configurer le mode kiosque

**Ce qu'on fait ici :** on crée un petit script (un fichier texte contenant une suite de commandes) qui va lancer Chromium directement en plein écran sur le dashboard CheckMK, et on configure le système pour lancer ce script automatiquement à chaque démarrage.

### 9.1 — Créer le fichier script

Cette commande ouvre un éditeur de texte dans le terminal, pour un fichier qui n'existe pas encore et qu'on va créer :
```bash
sudo nano /usr/bin/chromium-wallboard.sh
```
`nano` est un petit éditeur de texte simple, directement dans le terminal. Une fois la fenêtre ouverte, copie-colle ce contenu dedans (`Ctrl+Shift+V` pour coller) :
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

**Ce que fait chaque ligne du script**, en résumé :
- `URL="..."` : c'est l'adresse du dashboard qu'on veut afficher, stockée dans une variable pour pouvoir la modifier facilement plus tard sans toucher au reste
- `--kiosk` : lance Chromium en plein écran, sans aucune interface (barre d'adresse, onglets)
- `--noerrdialogs` / `--disable-infobars` / `--disable-session-crashed-bubble` : empêchent Chromium d'afficher des petites fenêtres d'avertissement (par exemple s'il a crashé une fois) qui viendraient polluer l'affichage
- `--check-for-update-interval=31536000` : espace les vérifications de mise à jour de Chromium à une fois par an, pour éviter que ça interfère avec l'affichage continu

**Pour enregistrer et fermer nano :** appuie sur `Ctrl + O` (la lettre O, pas un zéro) pour enregistrer, puis Entrée pour confirmer le nom du fichier, puis `Ctrl + X` pour quitter l'éditeur.

### 9.2 — Rendre ce script exécutable

Par défaut, un fichier texte n'est pas considéré comme un programme qu'on peut lancer. Cette commande lui donne "le droit de s'exécuter" :
```bash
sudo chmod +x /usr/bin/chromium-wallboard.sh
```

### 9.3 — Tester le script manuellement une première fois

Avant de le configurer pour un lancement automatique, teste-le directement pour voir si ça marche :
```bash
/usr/bin/chromium-wallboard.sh
```
Chromium devrait s'ouvrir en plein écran sur le dashboard CheckMK. Si tu vois bien ça, c'est gagné. Pour fermer, tu peux appuyer sur `Alt + F4`.

Si une erreur s'affiche à la place, note-la précisément — ça voudra dire qu'il faut corriger quelque chose avant de continuer.

### 9.4 — Faire en sorte que ce script se lance automatiquement à chaque ouverture de session

On va créer un petit fichier de configuration qui dit à Ubuntu "lance ce programme dès que la session utilisateur s'ouvre".

D'abord, crée le dossier prévu pour ce genre de fichiers (s'il n'existe pas déjà) :
```bash
mkdir -p ~/.config/autostart
```
`~` est un raccourci qui représente ton dossier personnel (`/home/checkmk-kiosk`). `mkdir -p` crée le dossier, et ne renvoie pas d'erreur s'il existe déjà.

Puis crée le fichier de configuration :
```bash
nano ~/.config/autostart/wallboard.desktop
```
Colle ce contenu :
```ini
[Desktop Entry]
Type=Application
Exec=/usr/bin/chromium-wallboard.sh
Hidden=false
X-GNOME-Autostart-enabled=true
Name=Wallboard CheckMK
Comment=Lance le kiosque CheckMK au démarrage de session
```
Enregistre (`Ctrl+O`, Entrée) et quitte (`Ctrl+X`).

✔️ **Test complet :** redémarre la machine (`sudo reboot`). À la remise en route, tu dois voir : le bureau Ubuntu s'ouvre automatiquement (grâce à l'autologin coché à l'installation), puis Chromium se lance en plein écran quelques secondes après, sur le dashboard CheckMK.

---

## 10. Empêcher l'écran de se mettre en veille

**Pourquoi :** par défaut, Ubuntu éteint l'écran et verrouille la session après un moment d'inactivité (pas de mouvement de souris/clavier) — exactement ce qu'on ne veut pas pour un panneau d'affichage permanent.

Ces commandes désactivent ces comportements. Chacune modifie un réglage précis du système :

```bash
gsettings set org.gnome.desktop.session idle-delay 0
```
→ Désactive le délai avant verrouillage automatique de session (`0` = jamais).

```bash
gsettings set org.gnome.desktop.screensaver lock-enabled false
```
→ Désactive le verrouillage d'écran par mot de passe même si l'écran de veille s'active.

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
```
→ Dit au système "ne fais rien de spécial (pas de mise en veille) quand la machine est inactive et branchée sur secteur".

```bash
gsettings set org.gnome.desktop.screensaver idle-activation-enabled false
```
→ Désactive complètement l'activation de l'économiseur d'écran.

```bash
gsettings set org.gnome.desktop.notifications show-banners false
```
→ Désactive les petites notifications qui pourraient apparaître en haut de l'écran (mises à jour disponibles, etc.) et venir gêner l'affichage.

Ces réglages s'appliquent immédiatement, pas besoin de redémarrer pour les tester.

---

## 11. Rendre le système résistant aux pannes

**Pourquoi :** le cahier des charges demande qu'en cas de coupure de courant ou de plantage du navigateur, tout reparte automatiquement sans intervention humaine. On va mettre en place deux protections.

### 11.1 — Un "surveillant" qui relance Chromium s'il plante

On crée un deuxième script qui, en boucle, vérifie toutes les 15 secondes si Chromium tourne encore ; s'il ne le voit plus, il le relance.

```bash
sudo nano /usr/bin/watchdog-wallboard.sh
```
Contenu à coller :
```bash
#!/bin/bash
while true; do
  if ! pgrep -x "chromium" > /dev/null; then
    /usr/bin/chromium-wallboard.sh &
  fi
  sleep 15
done
```

**Explication ligne par ligne :**
- `while true; do ... done` : une boucle infinie, qui répète ce qu'il y a dedans indéfiniment
- `pgrep -x "chromium"` : cherche si un programme nommé exactement "chromium" est actuellement en cours d'exécution
- `if ! ... > /dev/null` : "si ce programme n'est PAS trouvé" (le `!` veut dire "non")
- `then /usr/bin/chromium-wallboard.sh &` : si Chromium n'est pas trouvé, relance notre script (le `&` à la fin veut dire "lance ça en arrière-plan, sans bloquer la suite")
- `sleep 15` : attend 15 secondes avant de recommencer la vérification

Enregistre (`Ctrl+O`, Entrée) et quitte (`Ctrl+X`).

Rends-le exécutable, comme pour le premier script :
```bash
sudo chmod +x /usr/bin/watchdog-wallboard.sh
```

Fais-le se lancer automatiquement aussi, avec un deuxième fichier autostart :
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

### 11.2 — Redémarrage automatique après une coupure de courant

Ce réglage se fait dans le BIOS (le petit programme qui se lance avant même Ubuntu, propre à la carte mère).

1. Redémarre le PC
2. Appuie plusieurs fois sur **F2** dès que le logo Dell apparaît, pour entrer dans le BIOS
3. Cherche une option nommée **"AC Recovery"** ou **"Power On After Power Failure"** (l'emplacement exact varie selon la version du BIOS Dell, regarde dans les menus "Power Management" ou similaire)
4. Mets cette option sur **"Power On"** (ou "On")
5. Enregistre et quitte le BIOS (souvent la touche `F10`, avec confirmation)

Sans ce réglage, si le courant coupe puis revient, le PC resterait éteint tant que personne n'appuie physiquement sur le bouton d'allumage.

---

## 12. Créer le compte CheckMK dédié

**Pourquoi :** plutôt que d'utiliser le compte administrateur de CheckMK sur cet écran affiché en continu dans un lieu public, on crée un compte à droits limités, juste pour la consultation.

1. Depuis n'importe quel autre PC, ouvre `http://192.168.198.21/monitoring` et connecte-toi avec un compte admin CheckMK
2. Dans le menu, va dans **Setup > Users**
3. Clique sur "Ajouter un utilisateur" (ou équivalent selon la version)
4. Nom d'utilisateur : `kiosque`
5. Choisis un mot de passe solide, à noter dans votre gestionnaire de mots de passe professionnel
6. Dans le champ "Rôle", choisis un rôle en **lecture seule** (souvent appelé "guest" ou "monitoring" selon la version de CheckMK) — surtout pas "admin"
7. Enregistre

---

## 13. Sécuriser l'accès à distance

**Pourquoi :** le cahier des charges demande un accès SSH pour la maintenance à distance, sans avoir à se déplacer physiquement devant le PC.

> ✔️ **Simplifié grâce à l'IP fixe :** comme la machine a maintenant une adresse IP qui ne change jamais (section 6), tu peux te connecter directement avec cette IP, sans astuce particulière.

Installe le serveur SSH (le programme qui permet à d'autres machines de se connecter à distance à celle-ci) :
```bash
sudo apt install -y openssh-server
```

Active-le pour qu'il démarre automatiquement avec la machine :
```bash
sudo systemctl enable ssh
```

**Pour tester depuis un autre PC** (sur le même réseau), avec l'IP fixe donnée par l'admin réseau (section 6.3) :
```bash
ssh checkmk-kiosk@<IP_FIXE>
```
Ça va te demander le mot de passe du compte `checkmk-kiosk` que tu as créé à l'installation.

**Pour plus de sécurité**, on peut désactiver la connexion par simple mot de passe (et n'autoriser que par clé de sécurité — plus technique, à voir plus tard si besoin) :
```bash
sudo nano /etc/ssh/sshd_config
```
Cherche la ligne `PasswordAuthentication` et remplace sa valeur par `no`. Enregistre et quitte, puis redémarre le service :
```bash
sudo systemctl restart ssh
```
⚠️ Ne fais cette dernière partie que si tu as déjà mis en place une clé SSH de secours, sinon tu risques de te retrouver bloquée dehors.

---

## 14. Vérifier que tout fonctionne (recette)

Coche chaque point avec ton collègue et/ou l'admin réseau :

- [ ] Le PC est installé et branché à la TV du bureau du desk
- [ ] Au démarrage, sans que tu touches à rien, le bureau s'ouvre tout seul puis Chromium affiche le dashboard CheckMK en plein écran
- [ ] Aucune barre d'adresse, aucun onglet, curseur de souris invisible
- [ ] Test du certificat : `sudo snap refresh` ne renvoie pas d'erreur de certificat
- [ ] Tu débranches puis rebranches la prise électrique → la machine redémarre et réaffiche le dashboard toute seule
- [ ] L'adresse IP fixe reste bien identique après plusieurs redémarrages
- [ ] Après plusieurs heures, l'écran ne s'éteint jamais et ne se verrouille jamais
- [ ] Depuis un autre poste, la connexion SSH fonctionne via l'IP fixe
- [ ] Tu simules un plantage de Chromium avec la commande `pkill chromium` → il se relance tout seul en moins de 15 secondes
- [ ] Le compte `kiosque` sur CheckMK n'a accès qu'à la vue prévue, pas aux fonctions d'administration

---

*Document rédigé dans le cadre du BTS SIO SISR — Mairie de Saint-Égrève*
