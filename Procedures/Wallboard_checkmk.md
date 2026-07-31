# Déploiement d'un wallboard de supervision CheckMK

## 1. Contexte

Le service dispose d'une télévision inutilisée, non connectée au réseau. L'objectif est de la transformer en **panneau d'affichage de supervision** , affichant en permanence le tableau de bord CheckMK, 24h/24, sans interaction ni redémarrage intempestif.

> ℹ️ **Note**
> La solution repose entièrement sur des outils open source (Ubuntu Server, Openbox, Chromium), sans licence à payer, cohérent avec le choix de Checkmk Raw Edition (CRE) déjà en place sur ce projet.

## 2. Objectifs

- Afficher en boucle le tableau de bord de supervision CheckMK sur la TV du bureau
- Démarrage automatique, sans authentification, sans bureau visible
- Résilience 24h/24 : redémarrage automatique en cas de plantage du navigateur
- Aucune interaction requise (pas de souris/clavier après la mise en service)

## 3. Prérequis

| Élément | Détail |
|---|---|
| Matériel | Mini PC ou ancien PC (faible consommation recommandée) connecté à la TV en HDMI |
| OS | Ubuntu Server (pas de Desktop, pour rester léger) |
| Réseau | Connexion filaire Ethernet + IP fixe (ou réservation DHCP) |
| Accès | Serveur CheckMK accessible sur le réseau (`http://monitoring/<site>/check_mk/`) |

## 4. Procédure de déploiement

### Étape 1 — Installation du système et des paquets nécessaires

On installe Ubuntu Server (installation minimale), puis les paquets pour l'environnement graphique kiosque :

```bash
sudo apt update
sudo apt install -y xorg openbox chromium-browser unclutter x11-xserver-utils
```

- `xorg` + `openbox` : environnement graphique minimal, sans bureau complet
- `chromium-browser` : navigateur utilisé en mode kiosque
- `unclutter` : masque automatiquement le curseur de souris
- `x11-xserver-utils` : permet de désactiver la mise en veille de l'écran

### Étape 2 — Création du compte dédié à l'affichage

On crée un utilisateur `kiosk` séparé du compte d'administration, pour limiter les droits sur cette machine :

```bash
sudo adduser kiosk
```

### Étape 3 — Configuration de l'auto-login

On configure le démarrage pour connecter automatiquement le compte `kiosk` sur le terminal principal, sans mot de passe :

```bash
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d
sudo tee /etc/systemd/system/getty@tty1.service.d/override.conf <<EOF
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin kiosk --noclear %I \$TERM
EOF
```

### Étape 4 — Lancement automatique de l'environnement graphique

On ajoute dans `/home/kiosk/.bash_profile` :

```bash
if [ -z "$DISPLAY" ] && [ "$(tty)" = "/dev/tty1" ]; then
  startx
fi
```

### Étape 5 — Configuration d'Openbox (script de démarrage)

On crée `/home/kiosk/.config/openbox/autostart` avec le contenu suivant :

```bash
xset s off
xset s noblank
xset -dpms
unclutter -idle 0.5 -root &

while true; do
  chromium-browser --kiosk --noerrdialogs --disable-infobars \
    --incognito --no-first-run --disable-session-crashed-bubble \
    --disable-translate --autoplay-policy=no-user-gesture-required \
    "http://monitoring/<site>/check_mk/"
  sleep 5
done
```

> ⚠️ **Attention**
> La boucle `while true` est essentielle : elle relance automatiquement Chromium en cas de plantage, avec un délai de 5 secondes. Sans elle, un crash du navigateur laisserait un écran noir en permanence.

### Étape 6 — Configuration côté CheckMK

On privilégie une **vue dashboard** dédiée plutôt que la page d'accueil standard, en activant le rafraîchissement automatique intégré à CheckMK (propriété du dashboard). Cela évite de gérer le rechargement de page côté navigateur.

### Étape 7 — Sécurisation et stabilité de la machine

- Désactivation ou planification nocturne fixe des mises à jour automatiques (`unattended-upgrades`), pour éviter un redémarrage impromptu en journée
- Activation de SSH pour permettre une intervention à distance sans débrancher la TV
- Machine positionnée sur le réseau interne uniquement, sans exposition externe

> ✔ **Résultat attendu**
> Au démarrage : écran noir, puis affichage plein écran du tableau de bord CheckMK, sans barre d'adresse, sans possibilité d'interaction, avec redémarrage automatique du navigateur en cas de plantage.

## 5. Points de vigilance

- Vérifier la résolution native de la TV pour éviter un rendu flou du dashboard
- Prévoir une alimentation stable (onduleur si coupures fréquentes) pour éviter les redémarrages non propres
- Tester le comportement après une coupure secteur : la machine doit redémarrer et relancer le kiosque automatiquement sans intervention manuelle
