# Premiers pas avec NixOS (Ver. 1.0)

> Guide en cours d'écriture


## Introduction

Bienvenue, 

Ce guide a été écris dans l'optique d'aider les nouveaux utilisateurs de NixOS (mais pas de Linux) a acquérir un début de méthode pour configurer et surtout comprendre comment configurer leur système. 

Ce guide n'est pas un substitut à la documentation officielle, voyez-le tout au plus comme un tutoriel d'introduction.

Vous apprendrez a manipuler les outils de base, manipuler les fichiers de configurations du système, home-manager.
Nous installerons NixOS depuis le terminal sans utiliser l'installateur "calamares" fournit. 

Ci-dessous, un échantillon des choses que nous allons voir : 

- Installation : 
  - Partition racine **btrfs**
    - Pas de chiffrement
    - Deux sous-volumes
      - Racine */*
      - Répertoire utilisateur */home*
    - Compression *zstd*
  - Chargeur de démarrage **systemd-boot** (ou grub, au choix)
  - Ajout d'une animation au boot (plymouth)
  - Pipewire ou Pulseaudio (au choix)
  - Installation de drivers **GPU Nvidia / AMD** (machine fixe comme portable). 
  - Gestion de l'imprimante
  - Gestion du Bluetooth
  - Environnement de bureau
- Post Installation : 
  - Découverte et configuration de l'environnement utilisateur avec **Home Manager**
  - Ajout et configurations de flatpak & appimages
- Installer NixOS lorsque l'on a déjà un fichier de configuration existant. 

Toutes ces étapes seront expliqués en détails avec des exemples, d'ailleurs un lien vers la documentation est écris pour chaque section du guide.  
Pour des raisons évidentes, ce guide n'intégrera pas l'ensemble des cas de figure possibles, j'ai cependant essayé de documenter un maximum les configurations pour un usage courant. 

L'ensemble du guide a été écris et testé en pratique sur trois appareils différents : 

- MacBookAir M2           -> Machine virtuelle (utm), architecture arm64 (aarch64)
- Lenovo Legion 5 15ACH6H -> Bare Metal, architecture amd64
- PC Fixe                 -> Machine virtuelle (qemu), architecture amd64, Ryzen 5950X, 32Go RAM, RX6600

Pour chaque architecture, j'ai pris le soin d'écrire des configurations qui fonctionnent.
Il est cependant probable que je sois passé a côté d'erreurs, cependant rien qui ne devrait vous poser problème. 

Si vous avez des remarques, un problème, ouvrez une issue. 

# Installations de NixOS 

## 1. Pré-requis 

- Récupérez l'image iso d'installation adapté a l'architecture de votre système [ici](https://nixos.org/download) (j'utiliserais la version 23.11 durant ce tutoriel, il s'agit de la version stable actuellement disponible).
- Générez une usb bootable avec l'iso. (Prenez l'iso comprenant l'interface graphique)

## 2. Installation 

### 2.1 Réseau et disposition du clavier

En bootant sur l'iso, vous arriverez dans un environnement gnome, patientez qu'une fenêtre d'installation apparaisse. 


- Si vous utilisez le Wi-Fi, ouvrez le menu en haut à droite, configurez-le (attention clavier en qwerty au début) puis revenez a la fenêtre d'installation.
Si un message d'erreur apparaît dans la fenêtre d'installation, fermez-la puis ouvrez les applications gnome et ouvrez *Install*.

- Remplissez les premières pages de cette fenêtre pour configurer la disposition du clavier et arrêtez-vous lorsque l'on vous demande de configurer l'utilisateur. 

- Ouvrez l'application `Console` et écrivez la commande suivante pour passer en root (admininistrateur): 

```bash
sudo -i 
```

Prenez un instant pour vérifier que votre accès internet est fonctionnel : 

```bash
ping -c 3 google.com
```

### 2.2 Partitionnement 

- Repérez le disque sur lequel vous installerez NixOS
- Vous pouvez lister les différents supports de stockage reconnu avec : 

```bash
fdisk -l 
```

Selon la nature de votre support de stockage, la nomination de celui-ci peut changer (notez que X peut être remplacé par une lettre ou un chiffre): 
- SSD/HDD SATA : /dev/sdX
- (Machine virtuelle) : /dev/vdX
- SSD NVME : /dev/nvmeXnX 

Puisque je suis dans une machine virtuelle, j'ai `/dev/vda`, si vous avez un nom différents, adaptez les commandes en conséquence. 

Ci-dessous un tableau illustrant le partitionnement que nous allons faire : 

| Nom de la partition | Système de fichiers | Taille | Description | 
| --- | --- | --- | --- |
| /dev/**vda1** | fat32 | 512M | EFI System Partition |
| /dev/**vda2** | swap | 1G | Linux Swap |
| /dev/**vda3** | btrfs | RESTE | Linux Filesystem |

Sentez-vous libre de modifier la taille des partitions comme bon vous semble.

- `/dev/vda1` servira au **boot**
- `/dev/vda2` servira a gérer le **swap** 
- `/dev/vda3` servira de répertoire racine (**/**)

Pour partitionner, j'utilise `cfdisk`. 

```bash
cfdisk /dev/vda
```

- Sélectionnez `GPT`

- Créez une première partition de `512M`
- Créez une seconde  partition de `1G` 
- Créez une troisième partition sans spécifier de taille (prendra le reste de l'espace disponible). 
- Modifiez le type de la partition `/dev/vda1` pour `EFI System Partition`
- Modifiez le type de la partition `/dev/vda2` pour `Linux SWAP`
- Le type de la dernière partition est bien défini. 
- Cliquez sur `Write`, tapez `yes` puis quittez. 

### 2.3 Formatage

Les commandes suivantes formatent les partitions que nous venons de créer. 

La première partition sera formatée en `vfat32` (vous pouvez utiliser du `fat32` si vous le souhaitez, retirez simplement le `v`).
La seconde partition est formatée en `SWAP`, nous le montons directement après son formatage. 
La dernière partition est formatée en `BTRFS`. 

```bash
mkfs.vfat -F32 -n boot /dev/vda1
mkswap /dev/vda2 && swapon /dev/vda2
mkfs.btrfs /dev/vda3
```

### 2.4 Montage des partitions 

Dans l'odre : 
- Monter `/dev/vda3` dans le répertoire `/mnt`. 
- Créer un *sous-volume* de nom `@` dans `/mnt`
- Créer un *sous-volume* de nom `@home` dans `/mnt`
- Démonter `/dev/vda3` de `/mnt`
- Monter le *sous-volume* `@` de la partition `/dev/vda3` dans `/mnt` (on active également la compression **zstd** de niveau **3**)
- Créer les répertoires `boot` et `home` dans `/mnt`
- Monter le *sous-volume* `@home` de la partition `/dev/vda3` dans `/mnt/home` (on active également la compression **zstd** de niveau **3**)
- Monter la partition `/dev/vda1` dans `/mnt/boot`

Si vous souhaitez utiliser une compression différente de **3**, vous pouvez le remplacer par une valeur entre **1** et **15**. 
Notez qu'une valeur plus grande, implique une charge plus importante sur le cpu pour compresser les données. 
De manière générale, une valeur entre **1** et **3** est recommandée / raisonnable.

```bash
mount /dev/vda3 /mnt
btrfs sub cr /mnt/@
btrfs sub cr /mnt/@home
umount /mnt/
mount -o defaults,noatime,compress=zstd:3,subvol=@ /dev/vda3 /mnt
mkdir -p /mnt/{boot,home}
mount -o defaults,noatime,compress=zstd:3,subvol=@home /dev/vda3 /mnt/home
mount /dev/vda1 /mnt/boot
```

### 2.5 Génération d'un fichier de configuration 
[DOCUMENTATION](https://nixos.wiki/wiki/Nixos-generate-config)

La commande ci-dessous va générer une configuration de départ en se basant sur votre matériel. 
Les fichiers générés apparaitront dans `/mnt/etc/nixos/` respectivement `hardware-configuration.nix` et `configuration.nix`

```bash
nixos-generate-config --root /mnt
```

### 2.6 Le rôle des deux fichiers générés  

Le premier fichier `hardware-configuration.nix` gère : 

- Les partitions a monter et les paramètres qui leur sont associés 
- Les modules du noyau a charger

Comme mentionné dans l'entête du fichier, ce fichier ne doit pas être modifié. 
Il peut a tout moment être modifié par NixOS dans le futur. 

Une exception, si pour une raison ou une autre, votre partition racine, boot... a changé d'UUID, c'est pertinent. 

Si vous voulez consulter le fichier pour voir ce qui en ressors : 

```bash
nano /mnt/etc/nixos/hardware-configuration.nix
```

Le second fichier `configuration.nix` contiendra les configurations de votre système. 

Vous y déclarerez par exemple : 
- Les utilisateurs du systèmes, leur groupe...
- Les paquets a installer a l'échelle du système
- Les langues du clavier, interface...
- Configuration du Firewall
- Configuration d'imprimantes
- Drivers (gpu, wifi...)
- Monter des disques
- ...

### 2.7 Analyse du fichier de configuration généré

Attardons-nous sur le fichier `configuration.nix` généré. 

Ignorez un instant les déclarations et observez la structure du fichier : 

Elle est composé d'un entête qui commence par `{` et se termine avec `};`

```nix
{ config, lib, pkgs, ... };
```

Ensuite nous avons le **corps** de la configuration qui commence par `{` et se termine par `}` (pas de `;` comme vous pouvez le voir). 

```nix
{
  # Configurations
}
```

Lorsque nous ajouterons des configurations, c'est dans cette seconde accolade que nous les ajouterons. 

Gardez en tête que l'ordre de déclaration des déclarations n'a **aucune importance**, **l'indentation non plus**. 

Vous n'avez que deux choses a respecter : 

- Lorsque vous ouvrez une parenthèse, une accolade, crochet... vous devez forcément la refermer. 
- Les déclarations sont systématiquements suivis par un point-virgule.

Concernant l'indentation, un conseil, essayez de garder les instructions, les accolades, etc alignés ensemble.
Les erreurs dans la configuration sont suffisamment pénible a repérer, vous vous simplifierez la tâche si vous garder une structure organisé et propre. 

Dans un soucis de lisibilité, j'ai retiré les commentaires. Vous n'êtes pas obligé de le faire. 
Que vous le fassiez ou non n'a aucun incidence

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

#### 2.7.1 Qui fait quoi ? 

Cette extrait importe le fichier `hardware-configuration.nix` que nous avons vu tout a l'heure.
Ni plus, ni moins. 

```nix
imports = 
  [
    ./hardware-configuration.nix
  ];
```

On demande a nix d'utiliser **systemd-boot** en guise de chargeur de démarrage. 
La deuxième déclaration indique au chargeur de démarrage si il a l'autorisation de modifier les variables **EFI** (c'est assez vague, mais si cette ligne ne vous pose pas de problème, laissez-là).

[DOCUMENTATION](https://nixos.wiki/wiki/Bootloader)

```nix
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
```

Si vous préférez utilisez grub, remplacez ces lignes par :

```nix
  boot.loader.grub.enable = true;
  boot.loader.grub.efiSupport = true;
  boot.loader.grub.useOSProber = true; # Si vous souhaitez utiliser un dual-boot, cette ligne demande a grub de chercher d'autres systèmes d'exploitation (non pris en charge dans ce guide). 
```

Ici, les trois paramètres font respectivement : 
- Active la prise en charge du serveur X. 
- Définis (et installe) GDM comme gestionnaire de connexion.
- Définis (et installe) GNOME comme environnement de bureau. 

```nix
  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;
```

Concernant cette ligne, il s'agit d'un repère pour Nix. 
En effet, lors d'une mise a jour, il est possible que des éléments dans la configuration soit interprété différemment. 
Cette information permet a Nix de comprendre que le fichier utilise des déclarations provenant de la version **23.11**. 

En d'autres termes, ça assure la rétro-compatibilité de vortre configuration. 

```nix
  system.stateVersion = "23.11";
```

#### 2.7.2 Ajoutons notre configuration 

Les éléments marqués comme étant facultatif... sont facultatif, ne pas les ajouter n'aura aucune incidence sur votre système final.
C'est a vous de voir.

Lorsque vous aurez terminé de configurer votre fichier, passez un coup d'oeil [ici](https://github.com/NixOS/nixos-hardware/tree/master), ce dépôt contient des configurations supplémentaires pour prendre en charge le matériel. 

Mais juste avant, je vais vous montrer comment lire la documentation des déclarations. 

- Ouvrez cette [page](https://search.nixos.org/packages)
- Cliquez dans le menu **NixOS options** 
- A la ligne **Channel**, sélectionnez votre version (dans mon cas : *23.11*).
- Ecrivez par exemple "KDE", défilez la liste et cliquez sur *services.xserver.desktopManager.plasma5.enable*
- La documentation vous indique ce que fais la déclaration : 

> Enable the Plasma 5 (KDE 5) desktop environment.

- Il vous indique l'information qu'il attends. Ici par exemple il demande un "boolean" (*True* ou *False*) : 

**(Ne l'écrivez pas, c'est un exemple.)**

```nix
services.xserver.desktopManager.plasma5.enable = true;
```

Tout le long du guide, lorsque je parlerais de *déclaration*, je parlerais des "commandes" que l'on écris dans la configuration. 

##### Plymouth (facultatif)

Ajoutons **[Plymouth](https://fr.wikipedia.org/wiki/Plymouth_(informatique))** une application pour ajouter une animation de chargement au démarrage.

Ajoutez ces lignes où bon vous semble, en ce qui me concerne je préfère les ajouter a la suite de `boot.loader.[...]` pour la cohérence (puisque c'est lié au boot). 

La première ligne active `Plymouth` 
La seconde définit le thème "breeze". 

Si vous n'ajoutez pas cette seconde ligne, c'est le thème *bgrt* qui sera utilisé (comme indiqué [ici](https://search.nixos.org/options?channel=23.11&show=boot.plymouth.theme&from=0&size=50&sort=alpha_asc&type=packages&query=plymouth)).
Si jamais ce n'était pas clair, vous verrez de nombreuses configuration possible pour un logiciel mais souvent nous n'en utiliserons pas la moitié. 

- Le thème "**breeze**" affiche le logo nix + version avec une roue de chargement. 
- Le thème "**bgrt**" affiche le logo de la marque de votre appareil ainsi qu'une roue de chargement. 

```nix
boot.plymouth.enable = true;
boot.plymouth.theme = "breeze";
```

Votre fichier doit ressembler à quelque chose comme ça : 

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

##### Hostname (facultatif)

Le Hostname est le nom que l'on attribue a un appareil pour l'identifier sur un réseau. 

Ce paramètre est définis avec la [déclaration](https://search.nixos.org/options?channel=23.11&show=networking.hostName&from=0&size=50&sort=alpha_asc&type=packages&query=hostname) suivante : 

```nix
networking.hostName = "mydevicename";
```

S'il ne se voit pas attribuer de valeur, il prendra par défaut le nom de votre distribution (NixOS), comme mentionné dans la [documentation](https://search.nixos.org/options?channel=23.11&show=networking.hostName&from=0&size=50&sort=alpha_asc&type=packages&query=hostname).
N'utilisez pas d'underscore `_` dans le nom, comme indiqué dans la documentation. 

Notez que cette ligne est déjà écrite dans votre configuration (si vous n'avez pas supprimé les commentaires), supprimez simplement le caractère `#` en début de ligne. 

Le fichier : 

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

##### Network Manager

[Network Manager](https://wiki.archlinux.org/title/NetworkManager_(Fran%C3%A7ais)), est un outil permettant de gérer votre réseau. 
Puisque nous utilisons GNOME, nous l'activerons. 

Là aussi, la ligne est déjà écrite, vous pouvez la décommenter. 

```nix
networking.networkmanager.enable = true;
```

Notez qu'en activant cette option, vous ne pouvez pas utiliser la déclaration ci-dessous (incompatible ensemble) 

```nix
networking.wireless.enable = true;
```

Auquel cas, une erreur apparaîtra tout a l'heure lorsque nous installerons le système. 

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

##### Time Zone

Le fuseau horaire permet de déterminer l'heure et la date de votre système. 

Tel qu'écris dans votre configue, vous trouverez la ligne : 

```nix
time.timeZone = "Europe/Amsterdam";
```

Si vous vivez en France : 

```nix
time.timeZone = "Europe/Paris";
```

Si vous vivez ailleurs, voici une [liste des fuseaux horaires](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones), récupérer l'information correspondante dans la colonne *TZ Identifier*.

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

##### Configuration de la langue des menu, de l'interface

[Déclaration](https://search.nixos.org/options?channel=23.11&show=i18n.defaultLocale&from=0&size=50&sort=alpha_asc&type=packages&query=i18n.defaultLocale)

Pour les français : 

```nix
i18n.defaultLocale = "fr_FR.UTF-8";
```

Le format de l'argument est `language[_territory][.codeset][@modifier]`
Cliquez [ici](https://docs.moodle.org/403/en/Table_of_locales) pour une liste des langues possible.

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

##### Langue et disposition du clavier

Supprimez les lignes suivantes : 

```nix
# services.xserver.xkb.layout  = "us";
# services.xserver.xkb.options = "eurosign:e,caps:escape";
```

Ajoutez : 

```nix
services.xserver.xkb.layout  = "fr";
services.xserver.xkb.variant = "azerty";
```

[Déclaration ligne 1](https://search.nixos.org/options?channel=23.11&show=services.xserver.xkb.layout&from=0&size=50&sort=alpha_asc&type=packages&query=services.xserver.xkb)
[Déclaration ligne 2](https://search.nixos.org/options?channel=23.11&show=services.xserver.xkb.variant&from=0&size=50&sort=alpha_asc&type=packages&query=services.xserver.xkb)

Si vous avez un clavier mac, remplacez `azerty` par `mac`.

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

##### Prise en charge des imprimantes
[Documentation](https://nixos.wiki/wiki/Printing)
[Déclaration](https://search.nixos.org/options?channel=23.11&show=services.printing.enable&from=0&size=50&sort=alpha_asc&type=packages&query=printing)

Noux allons activer le service d'impression :

```nix
services.printing.enable = true;
```

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.printing.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

##### Configuration du son

Ici, vous avez différentes options, je vous présenterez les deux principales. 
Considérez que **pulseaudio** est l'ancienne méthode pour gérer le son, **pipewire** est la nouvelle. 

Pipewire est censé remplacer pulseaudio, la capture et la lecture de l'audio est pensé pour offrir une latence minimale. 
Plus d'info [ici](https://pipewire.org/)

Si vous ne savez pas quoi utiliser, commencez par essayer **pipewire**, si pour une raison ou une autre, vous avez des problèmes avec le son, basculez sur pulseaudio. N'ayez aucune crainte, il vous suffiera de supprimer les lignes de pipewire et les remplacer par celle de pulseaudio et appliquer la configuration. Nix s'occupera du reste. 


Pour **Pipewire** : 
[Documentation](https://nixos.wiki/wiki/PipeWire)

```nix
sound.enable = true;
hardware.pulseaudio.enable = false;
security.rtkit.enable = true;
services.pipewire.enable = true;
services.pipewire.alsa.enable = true;
services.pipewire.alsa.support32Bit = true;
services.pipewire.pulse.enable = true;
```

Pour **Pulseaudio** :
[Documentation](https://nixos.wiki/wiki/PulseAudio)

```nix
sound.enable = true;
hardware.pulseaudio.enable = true;
security.rtkit.enable = true;
```

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.printing.enable = true;

  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

##### Activation du trackpad pour les pc portables (facultatif)

[Déclaration](https://search.nixos.org/options?channel=23.11&show=services.xserver.libinput.enable&from=0&size=50&sort=alpha_asc&type=packages&query=services.xserver.libinput)

Comme indiqué dans la documentation de la déclaration, par défaut cette option est activée.
Notez également que les environnements de bureau active également cette option par défaut. 

```nix
services.xserver.libinput.enable = true;
```

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.printing.enable = true;

  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  services.xserver.libinput.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  system.stateVersion = "23.11";
}
```

##### Créons notre premier utilisateur

[Documentation](https://nixos.org/manual/nixos/stable/#sec-user-management)
[Déclaration](https://search.nixos.org/options?channel=23.11&show=users.users.%3Cname%3E.password&from=0&size=50&sort=relevance&type=packages&query=users.users.)


Par défaut, dans votre fichier de configuration, vous avez (pensez a décommentez le bloc de votre côté) : 

```nix
users.users.alice = {
  isNormalUser = true;
  extraGroups = [ "wheel" ];
  packages = with pkgs; 

  ];
};
```

On va prendre un instant pour décortiquer ce que fais le bloc. 

La première ligne `users.users.alice` indique a nix de créer un utilisateur du nom d'**alice**. 
Notez que le nom est en miniscule. 

La seconde [ligne](https://search.nixos.org/options?channel=23.11&show=users.users.%3Cname%3E.isNormalUser&from=0&size=50&sort=relevance&type=packages&query=users.users.%3Cname%3E) indique qu'il s'agit d'un véritable utilisateur (ça signifie qu'il sera attribué au groupe linux *user*, aura son propre espace de stockage **/home/alice**...) 

La troisième [ligne](https://search.nixos.org/options?channel=23.11&show=users.users.%3Cname%3E.extraGroups&from=0&size=50&sort=relevance&type=packages&query=users.users.%3Cname%3E) permet d'ajouter cet utilisateur a un ou plusieurs groupe. 
L'argument **wheel** indique que l'utilisateur fait parti du groupe pouvant utiliser **sudo** et exécuter des commandes nécessitant un privilège administrateur.

La dernière [ligne](https://search.nixos.org/options?channel=23.11&show=users.users.%3Cname%3E.packages&from=0&size=50&sort=relevance&type=packages&query=users.users.%3Cname%3E) permet d'indiquer quel logiciel installer a cet utilisateur uniquement (et donc indisponible aux autres). 

Vous n'avez qu'a écrire le nom du paquet a l'intérieur par exemple :

```nix
firefox
htop 
neofetch
```

La mise en forme n'a aucune importance, vous auriez pu l'écrire aussi : 

```nix
firefox htop neofetch
```

En "temps normal", nous ajouterions ici la liste des paquets a installer. 
Dans le cadre de ce guide, je vous apprendrais a utiliser `Home Manager` qui fait la même chose... en mieux ! 

Pour que notre utilisateur puisse manipuler les paramètres du réseau, nous ajouterons l'utilisateur au groupe `networkmanager`.
J'ai également ajouté les groupes `audio` et `video`.

Voici le bloc de code a écrire : 

- Remplacez `<yourusername>` par ce que vous voulez, utilisez des minuscules.
- Pour ceux qui le souhaite, j'ai ajouté une ligne `description`, vous pouvez y ajouter votre Nom et Prénom. 

```nix
users.users.<yourusername> = {
  isNormalUser = true;
  description = "Nom Prénom";
  extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
  packages = with pkgs; [

  ];
};
```

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.printing.enable = true;

  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  services.xserver.libinput.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  users.users.coraal = {
    isNormalUser = true;
    description = "CORAAL";
    extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
    packages = with pkgs; [

    ];
  };

  system.stateVersion = "23.11";
}
```

#### Paquets propriétaires

Pour installer des paquets propriétaires (steam, driver nvidia...) nous aurons besoin de l'autoriser : 

```nix
  nixpkgs.config.allowUnfree = true;
```

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.printing.enable = true;

  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  services.xserver.libinput.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  users.users.coraal = {
    isNormalUser = true;
    description = "CORAAL";
    extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
    packages = with pkgs; [

    ];
  };

  nixpkgs.config.allowUnfree = true;

  system.stateVersion = "23.11";
}
```

#### Checkpoint

Nous avons beaucoup modifié le fichier, prenez un instant pour sauvegarder si vous ne l'avez pas fait `CTRL + O`, `ENTRER`.

Prenez le temps de vérifier que vous n'ayez pas oublié un `;` à la fin des déclarations. 

##### Paquets a l'échelle du système

Nous allons maintenant ajouter des paquets qui seront disponible... pour tout le monde : 

```nix
  environment.systemPackages = [

  ];
```

Puisque nous avons utilisé btrfs, nous pourrions avoir besoin de `compsize` pour voir si la compression a été utile. 
Comme pour les paquets de l'utilisateur, on ajoute le nom dans la zone entre crochets : 

```nix
  environment.systemPackages = [
    pkgs.compsize
    pkgs.git
  ];
```

J'ai aussi ajouté `git`, nous en aurons besoin plus tard. 

- *Mais... il n'y a pas un problème là ?*
Bien vu. Ce guide est terrible long. 

Non le problème est ailleurs, pourquoi devons nous écrire "pkgs." devant le paquet mais pourquoi nous n'avons pas eu a le faire auparavant avec l'utilisateur ? 

C'est très simple. 
Dans nix, le nom d'un paquet est en réalité composé du préfixe `pkgs.`+`nom_du_paquet`. 
Dans le cas de l'utilisateur, nous avons utilisé sans le savoir un tour de passe-passe pour ne pas avoir a écrire le préfixe. 
Nous allons le refaire. 

Il suffit de modifier la déclaration : 

```nix
  environment.systemPackages = with pkgs; [

  ];
```

Nix lira cette déclaration comme : *pour chaque paquet dans la liste, ajouter "pkgs." devant chaque nom.*
Si au lieu de `pkgs` vous aviez écris `packages`, nix aurait lu : *pour chaque paquet dans la liste, ajouter *packages* devant chaque nom.*

Vous comprenez le principe.
On obtient : 

```nix
  environment.systemPackages = with pkgs; [
    compsize
    git
    rsync
    pciutils
    usbutils
  ];
```

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.printing.enable = true;

  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  services.xserver.libinput.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  users.users.coraal = {
    isNormalUser = true;
    description = "CORAAL";
    extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
    packages = with pkgs; [

    ];
  };

  environment.systemPackages = with pkgs; [
    compsize
    git
    rsync
    pciutils
    usbutils
  ];

  nixpkgs.config.allowUnfree = true;
  system.stateVersion = "23.11";
}
```

##### Configurons SSH 

Nous allons configurer l'accès SSH 

La première option active le serveur ssh, la seconde permet de modifier le port ssh (22 par défaut) pour 222. 

Vous n'êtes pas obligé de modifier le port, adaptez la configuration a vos besoin. 

```nix
services.openssh.enable = true;
services.openssh.settings.port = "222";
```

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.printing.enable = true;

  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  services.xserver.libinput.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  users.users.coraal = {
    isNormalUser = true;
    description = "CORAAL";
    extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
    packages = with pkgs; [

    ];
  };

  environment.systemPackages = with pkgs; [
    compsize
    git
    rsync
    pciutils
    usbutils
  ];

  services.openssh.enable = true;
  services.openssh.settings.port = "222";

  nixpkgs.config.allowUnfree = true;
  system.stateVersion = "23.11";
}
```

##### La compression BTRFS 

Pour finir (presque presque), nous allons activer la compression `zstd` pour btrfs. 
Nous aurions techniquement pu la modifier depuis `hardware-configuration.nix` mais si un jour ce fichier devait changer de forme ou de syntaxe, vous devriez recommencer. Nous ferons la modification dans `configuration.nix`. 

```nix
fileSystems = {
  "/".options =     [ "subvol=@" "compress=zstd:3" ];
  "/home".options = [ "subvol=@home" "compress=zstd:3" ];
};
```

Si vous avez déjà manipulé `fstab` sur d'autres distribution, les options que nous déclarons ici sont les mêmes. 

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
  
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.printing.enable = true;

  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  services.xserver.libinput.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  users.users.coraal = {
    isNormalUser = true;
    description = "CORAAL";
    extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
    packages = with pkgs; [

    ];
  };

  environment.systemPackages = with pkgs; [
    compsize
    git
    rsync
    pciutils
    usbutils
  ];

  services.openssh.enable = true;
  services.openssh.settings.port = "222";

  fileSystems = {
    "/".options =     [ "subvol=@" "compress=zstd:3" "noatime" "defaults" "discard"];
    "/home".options = [ "subvol=@home" "compress=zstd:3" "noatime" "defaults" "discard"];
  };

  nixpkgs.config.allowUnfree = true;
  system.stateVersion = "23.11";
}
```

J'ai ajouté les arguments **defaults**, **noatime** et **discard** qui respectivement : 
- monte le système de fichier avec des paramètres de base (rw, suid...)
- ne met pas a jour la date d'accès a un fichier a chaque fois qu'il est consulté (permet de réduire les écritures)
- active trim pour les ssd (le système indique au ssd qu'un bloc n'est plus utilisé)

Vous n'êtes pas obligé de les ajouter, votre système fonctionnera quand même. 
Plus d'info [ici](https://wiki.debian.org/fr/fstab)

##### Ajouter un kernel plus récent (facultatif)
[DOCUMENTATION](https://nixos.wiki/wiki/Linux_kernel)

Que ce soit parce que vous avez du matériel récent ou autre chose, vous avez la possibilité d'installer un noyau plus récent que celui par défaut :

```nix
  boot.kernelPackages = pkgs.linuxPackages_latest;
```

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];
 
  boot.kernelPackages = pkgs.linuxPackages_latest;

  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  services.printing.enable = true;

  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  services.xserver.libinput.enable = true;

  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  users.users.coraal = {
    isNormalUser = true;
    description = "CORAAL";
    extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
    packages = with pkgs; [

    ];
  };

  environment.systemPackages = with pkgs; [
    compsize
    git
    rsync
    pciutils
    usbutils
  ];

  services.openssh.enable = true;
  services.openssh.settings.port = "222";

  fileSystems = {
    "/".options =     [ "subvol=@" "compress=zstd:3" "noatime" "defaults" "discard"];
    "/home".options = [ "subvol=@home" "compress=zstd:3" "noatime" "defaults" "discard"];
  };

  nixpkgs.config.allowUnfree = true;
  system.stateVersion = "23.11";
}
```

##### Bilan 

Félicitations, vous venez de créer une première configuration. 
Elle n'est ni parfaite ni complète elle est même dégueulasse a lire, mais elle devrait suffire pour démarrer le système.

Nous prendons un moment pour améliorer ça plus tard. 

Sauvegardez votre fichier avec `CTRL+O` et quittez avec `CTRL+X` si vous ne l'avez pas encore fait. 

#### 2.7.3 Let ~~Him~~ Nix Cook ! 

Il est temps de vérifier que la configuration fonctionne.

```bash
nixos-install
```

Si votre configuration est correct, Nix devrait procéder à l'installation.
Si elle ne l'est pas, 

- Vérifiez que les déclarations aient bien un point-virgule à la fin. 
- Vérifiez l'orthographe des déclarations depuis la documentation.


La durée de l'installation dépendra entre de votre bande passante et de votre machine. 

Après un moment, nix vous demandera un mot de passe, rentrez le mot de passe pour l'utilisateur `Root`. 

À la fin, le message `Installation Complete` devrait apparaître, si c'est le cas : 

On va modifier le mot de passe utilisateur : 

```bash
nixos-enter --root /mnt
```

```bash
passwd <nom_de_votre_user>
```
ex : 

```bash
passwd coraal
```

```bash
exit
```


```bash
cd ; umount -R /mnt
```

Cliquez dans le menu en haut à droite puis sur éteindre. 
Débrancher votre usb puis démarrez votre système. 

## 3 Organisons la configuration 

### 3.1 Introduction

Nous venons de terminer l'installation, nous nous sommes assurés que le système démarre avec cette configuration. 
Il est maintenant temps d'introduire une façon d'écrire la configuration de manière plus organisé. 

Ouvrez `/etc/nixos/configuration.nix` avec un éditeur. 

```bash
nano /etc/nixos/configuration.nix
```

Votre configuration devrait ressembler à ça, j'ai ajouté quelques commentaires pour séparer les déclarations entre-elles : 

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];

  #/--------------------\# 
  #| ° Kernel Version ° |#
  #\--------------------/#
  boot.kernelPackages = pkgs.linuxPackages_latest;

  #/-----------------\# 
  #| ° Boot Loader ° |#
  #\-----------------/#
  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.canTouchEfiVariables = true;

  #/--------------------\# 
  #| ° Boot Animation ° |#
  #\--------------------/#
  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";

  #/----------------\# 
  #| ° Networking ° |#
  #\----------------/#
  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  #/----------------------\# 
  #| ° Time & Languages ° |#
  #\----------------------/#
  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  #/--------------------------\# 
  #| ° Keyboard Disposition ° |#
  #\--------------------------/#
  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  #/-------------\# 
  #| ° Printer ° |#
  #\-------------/#
  services.printing.enable = true;

  #/-----------\# 
  #| ° Sound ° |#
  #\-----------/#
  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  #/--------------\# 
  #| ° Trackpad ° |#
  #\--------------/#
  services.xserver.libinput.enable = true;

  #/-----------------------------------\# 
  #| ° Xserver & Desktop Environment ° |#
  #\-----------------------------------/#
  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  #/--------------------------\# 
  #| ° Users Configurations ° |#
  #\--------------------------/#
  # ------------ # 
  #/------------\#
  #| ° CORAAL ° |#
  #\------------/#
  users.users.coraal = {
    isNormalUser = true;
    description = "CORAAL";
    extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
    packages = with pkgs; [

    ];
  };

  #/---------------------\# 
  #| ° System Packages ° |#
  #\---------------------/#
  environment.systemPackages = with pkgs; [
    compsize
    git
    rsync
    pciutils
    usbutils
  ];

  #/------------------\# 
  #| ° Secure Shell ° |#
  #|   Configuration  |#
  #\------------------/#
  services.openssh.enable = true;
  services.openssh.settings.port = "222";

  #/-----------------\# 
  #| ° FileSystems ° |#
  #|   Mount Options |#
  #\-----------------/#
  fileSystems = {
    "/".options =     [ "subvol=@" "compress=zstd:3" "noatime" "defaults" "discard"];
    "/home".options = [ "subvol=@home" "compress=zstd:3" "noatime" "defaults" "discard"];
  };

  #/---------------------\# 
  #| ° Unfree Software ° |#
  #\---------------------/#
  nixpkgs.config.allowUnfree = true;

  #/-------------------------------\# 
  #| ° Nix Configuration Version ° |#
  #|   DO NOT MODIFY !!!           |#
  #\-------------------------------/#
  system.stateVersion = "23.11";
}
```

Vous vous en êtes probablement rendu compte, on retrouve très souvent des répétitions dans les appélations des déclarations.
Prenons l'exemple de la section **Boot Loader**, on retrouve un **préfixe** commun : 

```nix
  #/-----------------\# 
  #| ° Boot Loader ° |#
  #\-----------------/#

  boot.loader.systemd-boot.enable      = true;
  boot.loader.efi.CanTouchEfiVariables = true;

```

A deux reprises on retrouve ce `boot.loader.`. 
Si vous êtes familiers avec la programmation, vous savez que l'on évite un maximum de se répéter et c'est précisémment ce que l'on va faire. 

Une autre façon d'écrire ces deux déclarations est la suivante : 

```nix
  #/-----------------\# 
  #| ° Boot Loader ° |#
  #\-----------------/# 
 
  boot.loader = {
    systemd-boot.enable      = true;
    efi.CanTouchEfiVariables = true;
};
```

Un autre exemple avec la section **Boot Animation** : 

```nix
  #/--------------------\# 
  #| ° Boot Animation ° |#
  #\--------------------/#

  boot.plymouth.enable = true;
  boot.plymouth.theme = "breeze";
```

Il peut s'écrire aussi : 

```nix
  #/--------------------\# 
  #| ° Boot Animation ° |#
  #\--------------------/#

  boot.plymouth = {
    enable = true;
    theme = "breeze";
  };
```

Vous voyez l'idée ?
Et si vous avez été attentif, vous avez vu que la section **Boot Loader** et **Boot Animation** partagent un préfixe commun `boot`. 
Ce qui nous permet d'écrire : 

```nix
  boot = {
  loader = {
    systemd-boot.enable      = true;
    efi.canTouchEfiVariables = true;
  };
  plymouth = {
    enable = true;
    theme  = "breeze";
  };
};
```

La configuration devrait ressembler à ça (vous noterez que j'ai unifié *Boot Loader* et *Boot Animation*) : 

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];

  #/--------------------\# 
  #| ° Kernel Version ° |#
  #\--------------------/#
  boot.kernelPackages = pkgs.linuxPackages_latest;

  #/-----------------------------\# 
  #| ° Boot Loader & Animation ° |#
  #\-----------------------------/#

  boot = {
    loader = {
      systemd-boot.enable      = true;
      efi.canTouchEfiVariables = true;
    };
    plymouth = {
      enable = true;
      theme  = "breeze";
    };
  };

  #/----------------\# 
  #| ° Networking ° |#
  #\----------------/#
  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;

  #/----------------------\# 
  #| ° Time & Languages ° |#
  #\----------------------/#
  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  #/--------------------------\# 
  #| ° Keyboard Disposition ° |#
  #\--------------------------/#
  services.xserver.xkb.layout  = "fr";
  services.xserver.xkb.variant = "azerty";

  #/-------------\# 
  #| ° Printer ° |#
  #\-------------/#
  services.printing.enable = true;

  #/-----------\# 
  #| ° Sound ° |#
  #\-----------/#
  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire.enable = true;
  services.pipewire.alsa.enable = true;
  services.pipewire.alsa.support32Bit = true;
  services.pipewire.pulse.enable = true;

  #/--------------\# 
  #| ° Trackpad ° |#
  #\--------------/#
  services.xserver.libinput.enable = true;

  #/-----------------------------------\# 
  #| ° Xserver & Desktop Environment ° |#
  #\-----------------------------------/#
  services.xserver.enable = true;
  services.xserver.displayManager.gdm.enable   = true;
  services.xserver.desktopManager.gnome.enable = true;

  #/--------------------------\# 
  #| ° Users Configurations ° |#
  #\--------------------------/#
  # ------------ # 
  #/------------\#
  #| ° CORAAL ° |#
  #\------------/#
  users.users.coraal = {
    isNormalUser = true;
    description = "CORAAL";
    extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
    packages = with pkgs; [

    ];
  };

  #/---------------------\# 
  #| ° System Packages ° |#
  #\---------------------/#
  environment.systemPackages = with pkgs; [
    compsize
    git
    rsync
    pciutils
    usbutils
  ];

  #/------------------\# 
  #| ° Secure Shell ° |#
  #|   Configuration  |#
  #\------------------/#
  services.openssh.enable = true;
  services.openssh.settings.port = "222";

  #/-----------------\# 
  #| ° FileSystems ° |#
  #|   Mount Options |#
  #\-----------------/#
  fileSystems = {
    "/".options =     [ "subvol=@" "compress=zstd:3" "noatime" "defaults" "discard"];
    "/home".options = [ "subvol=@home" "compress=zstd:3" "noatime" "defaults" "discard"];
  };

  #/---------------------\# 
  #| ° Unfree Software ° |#
  #\---------------------/#
  nixpkgs.config.allowUnfree = true;

  #/-------------------------------\# 
  #| ° Nix Configuration Version ° |#
  #|   DO NOT MODIFY !!!           |#
  #\-------------------------------/#
  system.stateVersion = "23.11";
}
```

Avant de répéter ça sur le reste de la configuration, nous allons vérifier que notre configuration ne contienne pas d'erreur. 
Sauvegardez et quittez votre configuration.
Depuis le terminal saisissez :

```bash
sudo nixos-rebuild switch
```

La configuration va être vérifié par Nix et il appliquera la configuration (pas besoin de redémarrer pour le moment). 
La prochaine fois que vous redémarrerez, vous verrez une nouvelle entrée dans le boot loader, il s'agit d'une version de votre configuration, ça signifie qu'en cas de problème, vous pourrez repasser sur une ancienne version. 

```nix
  #/----------------\# 
  #| ° Networking ° |#
  #\----------------/#
  networking.hostName = "virtualmachinenixos";
  networking.networkmanager.enable = true;
```

Peut s'écrire : 

```nix
  #/----------------\# 
  #| ° Networking ° |#
  #\----------------/#
  networking = {
    hostName = "virtualmachinenixos";
    networkmanager.enable = true; 
  };
```

Si vous avez saisi le fonctionnement, répliquez ça jusqu'a la fin du fichier, pensez après chaque modification de rebuild votre configuration comme vous l'avez fait précédemment.  
Lorsque vous aurez finis, vous devriez obtenir quelque chose de similaire a ceci : 

```nix
{ config, lib, pkgs, ...}:

{
  imports = 
    [
      ./hardware-configuration.nix
    ];

  #/--------------------\# 
  #| ° Kernel Version ° |#
  #\--------------------/#
  boot.kernelPackages = pkgs.linuxPackages_latest;

  #/-----------------------------\# 
  #| ° Boot Loader & Animation ° |#
  #\-----------------------------/#

  boot = {
    loader = {
      systemd-boot.enable      = true;
      efi.canTouchEfiVariables = true;
    };
    plymouth = {
      enable = true;
      theme  = "breeze";
    };
  };

  #/----------------\# 
  #| ° Networking ° |#
  #\----------------/#
  networking = {
    hostName = "virtualmachinenixos";
    networkmanager.enable = true; 
  };

  #/----------------------\# 
  #| ° Time & Languages ° |#
  #\----------------------/#
  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "fr_FR.UTF-8";

  #/--------------------------\# 
  #| ° Keyboard Disposition ° |#
  #\--------------------------/#
  services.xserver.xkb = {
    layout  = "fr";
    variant = "azerty";
  };

  #/-------------\# 
  #| ° Printer ° |#
  #\-------------/#
  services.printing.enable = true;

  #/-----------\# 
  #| ° Sound ° |#
  #\-----------/#
  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire = {
    enable = true;
    alsa = {
      enable       = true;
      support32Bit = true;
    };
    pulse.enable   = true;
  };

  #/--------------\# 
  #| ° Trackpad ° |#
  #\--------------/#
  services.xserver.libinput.enable = true;

  #/-----------------------------------\# 
  #| ° Xserver & Desktop Environment ° |#
  #\-----------------------------------/#
  services.xserver = {
    enable = true;
    displayManager.gdm.enable   = true;
    desktopManager.gnome.enable = true;
  };

  #/--------------------------\# 
  #| ° Users Configurations ° |#
  #\--------------------------/#
  # ------------ # 
  #/------------\#
  #| ° CORAAL ° |#
  #\------------/#
  users.users.coraal = {
    isNormalUser = true;
    description = "CORAAL";
    extraGroups = [ "wheel" "networkmanager" "audio" "video" ];
    packages = with pkgs; [

    ];
  };

  #/---------------------\# 
  #| ° System Packages ° |#
  #\---------------------/#
  environment.systemPackages = with pkgs; [
    compsize
    git
    rsync
    pciutils
    usbutils
  ];

  #/------------------\# 
  #| ° Secure Shell ° |#
  #|   Configuration  |#
  #\------------------/#
  services.openssh = {
    enable = true;
    settings.port = "222";
  };

  #/-----------------\# 
  #| ° FileSystems ° |#
  #|   Mount Options |#
  #\-----------------/#
  fileSystems = {
    "/".options =     [ "subvol=@"     "compress=zstd:3" "noatime" "defaults" "discard"];
    "/home".options = [ "subvol=@home" "compress=zstd:3" "noatime" "defaults" "discard"];
  };

  #/---------------------\# 
  #| ° Unfree Software ° |#
  #\---------------------/#
  nixpkgs.config.allowUnfree = true;

  #/-------------------------------\# 
  #| ° Nix Configuration Version ° |#
  #|   DO NOT MODIFY !!!           |#
  #\-------------------------------/#
  system.stateVersion = "23.11";
}
```

Vérifiez votre configuration et cette fois redémarrez votre système pour s'assurer qu'il n'y a aucun problème avant de continuer.

```bash
sudo nixos-rebuild switch
```

## 4 Ajoutons les pilotes GPU

Nous allons nous attaquer a un morceau un peu plus difficile. 
Dans un premier temps, ouvrez la documentation et **lisez-la**.

Selon la génération de votre gpu et selon vos besoins, vous devrez ajouter ou non des déclarations supplémentaires. 

### 4.1 AMD
[Documentation AMD](https://nixos.wiki/wiki/AMD_GPU)

On demande a charger le module du noyau **AMDGPU**

```nix
boot.initrd.kernelModules = [ "amdgpu" ];
```

Et on demande a utiliser le pilote video **AMDGPU**

```nix
services.xserver.videoDrivers = [ "amdgpu" ];
```

#### Vulkan 

Ce paramètre est normalement activé par défaut par l'intermédiaire de **MESA**. 

```nix
hardware.opengl.driSupport = true; 
hardware.opengl.driSupport32Bit = true;
```

Pour les architectures les plus anciennes comme *Southern Islands* ou *Sea Islands*, vous devez ajouter un paramètre supplémentaire :

#### Southern Islands

```nix
boot.kernelParams = [ "radeon.si_support=0" "amdgpu.si_support=1" ];
```

#### Sea Islands

```nix
boot.kernelParams = [ "radeon.cik_support=0" "amdgpu.cik_support=1" ];
```

### 4.2 NVIDIA
[Documentation NVIDIA](https://nixos.wiki/wiki/Nvidia)

```nix
  hardware.opengl = {
    enable = true;
    driSupport = true;
    driSupport32Bit = true;
  };
  services.xserver.videoDrivers = ["nvidia"];

  hardware.nvidia = {

    # Modesetting is required.
    modesetting.enable = true;

    powerManagement.enable = false;
    powerManagement.finegrained = false;

    # https://github.com/NVIDIA/open-gpu-kernel-modules#compatible-gpus 
    open = false; # 3000 Series and + = false

    nvidiaSettings = true;

    package = config.boot.kernelPackages.nvidiaPackages.stable;
  };
```

Pour les pc portables : 

`lshw` -> `sudo lshw -c display`

Nvidia : pci : 01:00.0
AMD    : pci : 06:00.0

```nix
	hardware.nvidia.prime = {
		# Make sure to use the correct Bus ID values for your system!
		amdBusId = "PCI:1:0:0";
		nvidiaBusId = "PCI:06:0:0";
	};
```

```bash
sudo nixos-rebuild switch 
```

Si vous avez une erreur lié a la licence, vous devrez probablement ajouter : 

```nix
 nixpkgs.config.nvidia.acceptLicense = true;
```

Redémarrez votre système.


## 4 Embrassez le minimalisme

En vous connectant a votre session la première fois, vous avez sans doute remarqué la présence d'un grand nombre d'applications, que vous n'utiliserez probablement jamais. 

Nous avons ici, l'environnement de bureau GNOME. Au cas où vous ne le sauriez pas déjà, un environnement de bureau, c'est une collection d'applications installés ensemble... et qui forme un environnement. 
Parmi cette collection, tout les paquets ne nous intéresse pas. 

Pour supprimer un paquet de l'environnement, on a besoin d'ajouter une déclaration `environment.gnome.excludePackages` suivis d'une liste des paquets a supprimer :


```nix
    environment.gnome.excludePackages = (with pkgs; [
      gnome-photos
      gnome-tour
      gedit
      epiphany
      evince

    ]) ++ (with pkgs.gnome; [
      cheese
      gnome-music
      gnome-terminal
      geary
      gnome-maps
      gnome-characters
      totem
      tali
      iagno
      hitori
      atomix
    ]);
```

J'attire votre attention sur l'écriture un peu particulière bien que familière par rapport a ce que nous avons vu jusqu'a présent. 

Durant l'installation depuis l'image ISO, je vous avais parlé de `with pkgs;` qui permettait en quelques sortes d'ajouter automatiquement le préfixe `pkgs.` devant chaque nom de paquet.

Ici, nous avons quelque chose de similaire sauf que l'on a cette fois `with pkgs;` et `with pkgs.gnome`. 
Il s'agit d'une "optimisation" de l'écriture (similaire a ce que l'on a fait plus tôt) et qui permet d'éviter de recopier les préfixes : 

- `pkgs.` dans la première partie de la liste
- `pkgs.gnome` dans la seconde partie de la liste

[gedit](https://search.nixos.org/packages?channel=23.11&show=gedit&from=0&size=50&sort=relevance&type=packages&query=gedit)
[gnome-music](https://search.nixos.org/packages?channel=23.11&show=gnome.gnome-music&from=0&size=50&sort=relevance&type=packages&query=gnome-music)

Sans ces deux "optimisations", nous aurions du écrire : 

```nix
    environment.gnome.excludePackages = [
      pkgs.gnome-photos
      pkgs.gnome-tour
      pkgs.gedit
      pkgs.epiphany
      pkgs.evince
      pkgs.gnome.cheese
      pkgs.gnome.gnome-music
      pkgs.gnome.gnome-terminal
      pkgs.gnome.geary
      pkgs.gnome.gnome-maps
      pkgs.gnome.gnome-characters
      pkgs.gnome.totem
      pkgs.gnome.tali
      pkgs.gnome.iagno
      pkgs.gnome.hitori
      pkgs.gnome.atomix
    ];
```

Pour voir quels applications font partis de l'environnement, rendez-vous sur cette [page](https://search.nixos.org/packages), écrivez **gnome** dans la recherche. Cliquez sur un des résultats puis sur **NixOS Configuration** pour voir le nom attendu dans la configuration. 

## 5 Configurons notre environnement 

Vous l'avez vu, nous avons la possibilité de configurer notre système avec `configuration.nix`. 
Nous pourrions même configurer notre environnement utilisateur depuis ce fichier. 
Mais ça pose deux problèmes : 

- Une configuration utilisateur, ça prends (ou ça peut prendre) beaucoup de place, si vous devez gérer plusieurs utilisateur, la configuration du système va gonfler en conséquence.
Ce n'est pas pratique a gérer.

- L'utilisateur doit avoir les droits administrateurs pour modifier la configuration même si c'est celle de son propre environnement.
Imaginez qu'un utilisateur non admin, ait besoin d'installer des apps, comment fait-il ? 

En utilisant **Home Manager** on peut séparer la configuration nix et celle de l'utilisateur. 
Ce dernier pourra même modifier son environnement depuis home manager sans passer par sudo.

## 6 Home Manager

La première chose a faire, c'est d'ajouter **Home Manager** dans les référentiels nix (pour qu'il sache où le chercher pour l'installer).

```bash
sudo nix-channel --add https://github.com/nix-community/home-manager/archive/release-23.11.tar.gz home-manager
sudo nix-channel --update
```

Installez **Home-Manager** : 

```bash
nix-shell '<home-manager>' -A install
```

Cette commande va installer et ajouter un fichier de configuration dans `~/.config/home-manager/home.nix`. 
Ce fichier contiendra toutes les configurations de l'environnement de **votre** utilisateur.

Nous allons voir quelques exemple de ce qu'il est possible de faire. 

### 6.1 Découverte du fichier

Comme pour `configuration.nix`, nous allons prendre quelques instants pour voir comment le fichier est construit.

```bash
nano ~/.config/home-manager/home.nix
```

Sans surprise, le fichier est très similaire : 

```nix
{ config, pkgs ...};

{
    # [...]
}
```

Concernant les déclarations qui y sont écrites : 

```nix
{ config, pkgs ...};

{
  home.username = "myusername";            # --> Nom de votre utilisateur     <--  // Ne pas modifier
  home.homeDirectory = "/home/myusername"; # --> L'emplacement de vos données <--  // Ne pas modifier

  home.stateVersion = "23.11";             # --> Version de nixos lorsque vous avez généré votre configuration 

  home.packages = {
                                           # --> C'est ici que vous écrirez, le nom des packages que vous souhaitez pour votre utilisateur.
  };

  home.sessionVariables = {
    EDITOR = "neovim";                     # --> C'est ici que vous écrirez le programme qui sera l'éditeur par défaut, vous avez le choix entre "nano", "vi", "vim", "neovim", "emacs"...
  };
  programs.home-manager.enable = true;     # --> Cette ligne permet d'activer home-manager, laissez là sur "true".
}
```

### 6.2 Ajout de paquets pour l'utilisateur

Commençons simple et ajoutons des paquets a destination de l'utilisateur. 

Et nous en profiterons pour modifier le bloc permettant de gérer les paquets pour ne pas avoir a écrire le préfixe `pkgs.` devant chaque paquets.

```nix
  home.packages = with pkgs;{
    vesktop
    firefox
    anki-bin
    tree
    git
    neofetch
    wget
    curl
    neovim
    remmina
    obs-studio
    # ...
  };
```

Si vous avez besoin d'installer des paquets non-libre (discord par exemple), ajoutez : 

```nix
nixpkgs.config.allowUnfree = true; 
```

Lorsque vous avez ajouté la liste de vos paquets, sauvegardez et quittez le fichier puis : 

```bash
home-manager switch
```

Vos paquets sont maintenant installés. 
Notez que même si les paquets n'apparaissent (pas encore) dans votre liste d'applications gnome, elles sont bien présente. 
Lorsque vous redémarrez, les icônes appraitront. 

Bien, vous allez maintenant apprendre a configurer vos applications et y intégrer les configurations dans `home.nix`.
Il existe plusieurs méthodes de configuration possibles : 
En utilisant un module, les flakes (que l'on a pas encore abordé) ou encore avec le fichier de config. --- A REFORMULER ---

Commençons avec un module. 

### 6.3 Configuration de Git (module)

Lorsque l'on utilise git pour gérer le versionning d'un projet, nous devons configurer trois choses : 

- Le nom de l'utilisateur / Pseudonyme
- L'email de l'utilisateur
- Le nom de la branche par défaut lorsque l'on initialise un dépôt (a la création d'un dépôt). 

Dans d'autres distributions, nous aurions configuré git comme suit : 

```bash
git config user.name  "John Smith"
git config user.email "johnsmith@wanted.domain"
git config init.defaultBranch "main"
```

Oubliez la troisième option `defaultBranch` pour le moment. 

Ouvrez cette [page](https://nix-community.github.io/home-manager/options.xhtml) et cherchez `programs.git` 

Vous retrouverez plusieus déclaration pour configurer git : 

- `programs.git.enable`
- `programs.git.userName`
- `programs.git.userEmail`
- `programs.git.package`
- `programs.git.aliases`
- `programs.git.attributes`
- `programs.git.ignores`
- ...

Dans notre cas, nous avons besoin d'activer git, ajouter un nom et un mail (l'option pour la branch n'existe pas a ma connaissance dans ce module) : 

```nix
  programs.git = {
    enable    = true;
    userName  = "John Smith";
    userEmail = "johnsmith@wanted.domain"
  };
```

Rien de plus. 

Sauvegardez...

```bash
home-manager switch
```

Patientez quelques instants, écrivez ensuite : 

```bash
git config --get user.name
git config --get user.email
```
Normalement les informations affichés devraient correspondre avec celle que vous venez de configurer. 

### 6.4 Configuration Git (config file)

Voyons une deuxième méthode pour configurer git, nous allons écrire le fichier de configuration directement au sein de notre fichier `home.nix` et **Home Manager** se chargera de copier le contenu de la configuration où il faut.

Git pour stocker les informations de configuration utilise un fichier `~/.gitconfig`. 

Si vous ouvrez ce fichier

```bash
nano ~/.gitconfig
```

Vous obtenez quelque chose comme : 

Vous pourriez avoir plus ou moins d'option, ça n'a pas d'importance. 

```.gitconfig
[user]
  name = "John Smith"
  email = "johnsmith@wanted.domain"
[init]
  defaultBranch = "main"
```

Pour l'ajouter a **Home Manager**, ouvrez de nouveau `home.nix` et ajoutez : 

```nix
  home.file.".gitconfig".text = ''
    [user]
      name = "John Smith"
      email = "johnsmith@wanted.com"
    [init]
      defaultBranch = "main"
  '';
```

Comme vous pouvez le constater en comparant le fichier *.gitconfig* et le nouveau bloc de *home.nix*, la seule différence est que l'on a une déclaration délimite la configuration avec au début : 

```nix
home.file.".gitconfig".text = ''
```
et a la fin : 

```nix
  '';
```

Dans la déclaration d'ouverture, `.gitconfig` fait référence au nom du fichier de configuration. Il indique a Home Manager où placer la configuration.  

Pour résumer, vous devez indiquer : 
- Le nom du fichier de configuration
- L'emplacement où "coller" la configuration 
- Le contenu de la configuration

