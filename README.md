* * *

# Installation d'Arch Linux en Dual Boot avec Windows 10 (UEFI)

Ce guide détaille l'installation d'Arch Linux en dual boot sur un système avec Windows 10 déjà installé en mode UEFI. **Une sauvegarde complète de vos données est impérative avant de commencer.**

## **0\. Pré-requis et Préparation Initiale**

1.  **Sauvegarder vos données :** C'est la priorité absolue. Sauvegardez tous vos documents, photos, vidéos, etc., sur un disque dur externe ou un service de cloud.
2.  **Préparer l'espace disque sous Windows 10 :**
    - Démarrez sous Windows 10.
    - Appuyez sur `Win + X` et sélectionnez "Gestion des disques".
    - Identifiez la partition où Windows est installé (généralement `C:`).
    - Faites un clic droit sur cette partition et choisissez "Réduire le volume...".
    - Entrez la quantité d'espace que vous souhaitez allouer à Arch Linux (ex: 50 Go = 51200 Mo, 100 Go = 102400 Mo). Laissez un espace suffisant pour Arch et vos futures données.
    - Vous verrez un "Espace non alloué" apparaître. Ne le formatez pas pour l'instant.
3.  **Désactiver le Démarrage rapide (Fast Startup) de Windows :**
    - Ouvrez le "Panneau de configuration".
    - Recherchez "Options d'alimentation".
    - Cliquez sur "Choisir l'action des boutons d'alimentation".
    - Cliquez sur "Modifier des paramètres actuellement non disponibles".
    - Décochez la case "Activer le démarrage rapide (recommandé)".
    - Enregistrez les modifications.
4.  **Désactiver Secure Boot (si activé) :** Accédez au BIOS/UEFI de votre ordinateur (généralement F2, F10, F12, Suppr au démarrage) et désactivez l'option "Secure Boot".
5.  **Désactiver BitLocker (si activé) :** Si votre disque Windows est chiffré avec BitLocker, désactivez-le temporairement.
6.  **Télécharger l'image ISO d'Arch Linux :** Rendez-vous sur le site officiel d'Arch Linux (archlinux.org/download/) et téléchargez la dernière image ISO.
7.  **Créer une clé USB bootable :** Utilisez un outil comme Rufus (Windows) ou Etcher (multi-plateforme) pour graver l'ISO sur une clé USB d'au moins 2 Go.

* * *

## **1\. Démarrer et Vérifier le Système**

1.  **Démarrer sur la clé USB d'installation d'Arch Linux :**
    - Redémarrez votre ordinateur.
    - Au démarrage, appuyez sur la touche appropriée (F2, F10, F12, Suppr, etc.) pour accéder au menu de démarrage ou au BIOS/UEFI.
    - Sélectionnez votre clé USB comme périphérique de démarrage.
    - Vous devriez arriver sur l'invite de commande d'Arch Linux (`root@archiso ~#`).
2.  **Vérifier le mode de démarrage (UEFI) :**
    - Vérifiez que vous êtes bien en mode UEFI (indispensable pour le dual boot avec Windows 10 moderne) :
        
        ```bash
        ls /sys/firmware/efi/efivars
        ```
        
        Si des fichiers sont listés, vous êtes en mode UEFI. Si la commande renvoie une erreur ou que le répertoire est vide, redémarrez et assurez-vous de bien booter la clé USB en mode UEFI.
3.  **Vérifier la connexion Internet :**
    - **Ethernet :** La connexion est souvent automatique. Vérifiez avec : `ping -c 2 google.com`.
    - **Wi-Fi :**
        
        ```bash
        iwctl
        device list # Notez le nom de votre carte Wi-Fi, ex: wlan0
        station <nom_carte> scan
        station <nom_carte> get-networks
        station <nom_carte> connect <SSID> # Entrez le mot de passe si demandé
        exit # Quittez iwctl
        ```
        
    - Si votre réseau nécessite une configuration IP statique (cas moins courant) :
        
        ```bash
        ifconfig eno16777736 192.168.1.52 netmask 255.255.255.0 # Adaptez l'interface et les adresses
        route add default gw 192.168.1.1 # Adaptez la passerelle
        echo "nameserver 8.8.8.8" >> /etc/resolv.conf # DNS Google
        ```
        
    - Vérifiez à nouveau la connexion : `ping -c 2 google.com`.
4.  **Mettre à jour l'horloge système :**
    
    ```bash
    timedatectl set-ntp true
    ```
    

* * *

## **2\. Partitionnement et Formatage**

1.  **Identifier votre disque dur :**
    
    ```bash
    lsblk
    fdisk -l
    ```
    
    Notez le nom de votre disque principal (ex: `/dev/sda` ou `/dev/nvme0n1`).
2.  **Lancer l'utilitaire de partitionnement :**
    
    ```bash
    cfdisk /dev/sda # Remplacez /dev/sda par votre disque principal
    ```
    
    - Utilisez les flèches pour naviguer et créer les partitions dans l'espace "Free space" non alloué que vous avez préparé depuis Windows.
    - **Créez les partitions pour Arch :**
        - **Partition Racine (`/`) :** Créez une nouvelle partition, donnez-lui une taille (ex: 40G-60G), type `Linux filesystem`.
        - **Partition Swap (Optionnel mais recommandé) :** Créez une nouvelle partition, taille (ex: 4G ou plus selon votre RAM), type `Linux swap`.
        - **Partition Home (`/home`, optionnel mais recommandé) :** Si vous avez de l'espace restant, créez une nouvelle partition avec le reste de l'espace, type `Linux filesystem`.
    - Écrivez les modifications (`Write`), puis quittez (`Quit`).
    - Vérifiez le nouveau tableau de partitions : `fdisk -l`. Notez les noms de vos nouvelles partitions (ex: `/dev/sda3` pour root, `/dev/sda4` pour swap, etc.).
3.  **Formater les partitions Arch :**
    
    ```bash
    mkfs.ext4 /dev/sdaX_root # Remplacez X_root par votre partition racine Arch (ex: sda3)
    mkswap /dev/sdaX_swap # Remplacez X_swap par votre partition swap (ex: sda2)
    ```
    
    - Si vous avez une partition `/home` distincte :
        
        ```bash
        mkfs.ext4 /dev/sdaX_home # Remplacez X_home par votre partition home
        ```
        
    - **N'OUBLIEZ PAS : NE PAS FORMATER LA PARTITION EFI DE WINDOWS !**
4.  **Activer la partition Swap :**
    
    ```bash
    swapon /dev/sdaX_swap # Remplacez X_swap par votre partition swap (ex: sda2)
    ```
    

* * *

## **3\. Montage des Partitions**

1.  **Monter la partition Racine :**
    
    ```bash
    mount /dev/sdaX_root /mnt # Remplacez X_root par votre partition racine (ex: sda3)
    ```
    
2.  **Créer le point de montage pour la partition EFI :**
    
    ```bash
    mkdir -p /mnt/boot/efi
    ```
    
3.  **Monter la partition EFI EXISTANTE de Windows :**
    
    - **TRÈS IMPORTANT :** Identifiez correctement la partition EFI de Windows. C'est généralement une petite partition (100-500 Mo) formatée en FAT32, souvent la première partition du disque (ex: `/dev/sda1` ou `/dev/nvme0n1p1`).
    - Vérifiez avec `lsblk -f`.

    
    ```bash
    mount /dev/sdaX_efi /mnt/boot/efi # Remplacez X_efi par la partition EFI de Windows (ex: sda1)
    ```
    
4.  **Monter la partition Home (si créée) :**
    
    ```bash
    mkdir /mnt/home
    mount /dev/sdaX_home /mnt/home # Remplacez X_home par votre partition home
    ```
    

* * *

## **4\. Installation du Système de Base**

1.  **Mettre à jour la liste des miroirs (facultatif mais recommandé) :**
    
    ```bash
    nano /etc/pacman.d/mirrorlist
    ```
    
    Déplacez les serveurs les plus proches de vous en haut de la liste pour des téléchargements plus rapides.
2.  **Installer les paquets de base :**
    
    ```bash
    pacstrap /mnt base base-devel linux linux-firmware nano vim
    ```
    
    - `base` : système de base d'Arch
    - `base-devel` : outils de développement essentiels
    - `linux` : le noyau Linux
    - `linux-firmware` : microprogrammes pour le matériel
    - `nano` et `vim` : éditeurs de texte
3.  **Générer le fichier fstab :**
    
    ```bash
    genfstab -U -p /mnt >> /mnt/etc/fstab
    ```
    
    - Vérifiez le contenu de `fstab` pour s'assurer que toutes les partitions sont correctement listées, y compris la partition EFI :
        
        ```bash
        cat /mnt/etc/fstab
        ```
        

* * *

## **5\. Configuration du Système**

1.  **Chrooter dans le nouveau système Arch :**
    
    ```bash
    arch-chroot /mnt
    ```
    
    Vous êtes maintenant dans votre nouvelle installation Arch.
    
2.  **Définir le nom d'hôte :**
    
    ```bash
    echo "arch-pc" > /etc/hostname # Remplacez par le nom de votre choix
    ```
    
    - Ajoutez des entrées dans `/etc/hosts` :
        
        ```bash
        nano /etc/hosts
        ```
        
        ```
        127.0.0.1       localhost
        ::1             localhost
        127.0.1.1       arch-pc.localdomain arch-pc # Utilisez votre nom d'hôte
        ```
        
3.  **Configurer la localisation (Langue) :**
    
    ```bash
    nano /etc/locale.gen
    ```
    
    Décommentez les lignes suivantes (supprimez le `#` devant) selon votre région :
    
    - **Pour le Québec (Canada) :**
        
        ```
        fr_CA.UTF-8 UTF-8
        en_US.UTF-8 UTF-8
        ```
        
    - **Pour la France :**
        
        ```
        fr_FR.UTF-8 UTF-8
        en_US.UTF-8 UTF-8
        ```
        
    - **Pour la Belgique :**
        
        ```
        fr_BE.UTF-8 UTF-8
        en_US.UTF-8 UTF-8
        ```
        
    - **Pour la Suisse :**
        
        ```
        fr_CH.UTF-8 UTF-8
        de_CH.UTF-8 UTF-8 # Si vous utilisez aussi l'allemand
        it_CH.UTF-8 UTF-8 # Si vous utilisez aussi l'italien
        en_US.UTF-8 UTF-8
        ```
        
    
    Générez les locales :
    
    ```bash
    locale-gen
    ```
    
    Créez le fichier de configuration de la locale (choisissez la ligne correspondant à votre sélection ci-dessus) :
    
    - **Pour le Québec :** `echo LANG=fr_CA.UTF-8 > /etc/locale.conf`
    - **Pour la France :** `echo LANG=fr_FR.UTF-8 > /etc/locale.conf`
    - **Pour la Belgique :** `echo LANG=fr_BE.UTF-8 > /etc/locale.conf`
    - **Pour la Suisse :** `echo LANG=fr_CH.UTF-8 > /etc/locale.conf`
    
    Appliquez la locale pour la session courante :
    
    ```bash
    export LANG=<VOTRE_LOCALE_CHOISIE> # ex: export LANG=fr_CA.UTF-8
    ```
    
4.  **Configurer le fuseau horaire :**
    
    ```bash
    ls /usr/share/zoneinfo/ # Pour trouver votre fuseau horaire, ex: Europe/Paris, Europe/Brussels, Europe/Zurich, America/Montreal
    ln -s /usr/share/zoneinfo/<VOTRE_REGION>/<VOTRE_VILLE> /etc/localtime # Adaptez à votre fuseau
    hwclock --systohc --utc
    ```
    
    - **Pour le dual boot avec Windows (TRÈS IMPORTANT) :**
        
        ```bash
        timedatectl set-local-rtc 1 --adjust-system-clock
        ```
        
        Cela permet d'éviter les problèmes de décalage horaire entre Windows (qui utilise l'heure locale sur le RTC) et Arch (qui utilise par défaut l'UTC).
5.  **Activer les dépôts Multilib (Optionnel) :**
    
    ```bash
    nano /etc/pacman.conf
    ```
    
    Décommentez les lignes suivantes :
    
    ```
    [multilib]
    Include = /etc/pacman.d/mirrorlist
    ```
    
    Synchronisez les paquets après cette modification :
    
    ```bash
    pacman -Syu
    ```
    
6.  **Définir le mot de passe root :**
    
    ```bash
    passwd
    ```
    
    Choisissez un mot de passe fort.
    
7.  **Créer un nouvel utilisateur (non-root) :**
    
    ```bash
    useradd -m -g users -G wheel,storage,power -s /bin/bash votrenomutilisateur # Remplacez par votre nom d'utilisateur
    passwd votrenomutilisateur
    chage -d 0 votrenomutilisateur # Force le changement de mot de passe au premier login (bonne pratique de sécurité)
    ```
    
8.  **Configurer `sudo` :**
    
    ```bash
    pacman -S sudo
    visudo # Ouvre le fichier /etc/sudoers de manière sécurisée
    ```
    
    Décommentez la ligne suivante pour permettre aux utilisateurs du groupe `wheel` d'utiliser `sudo` :
    
    ```
    %wheel ALL=(ALL) ALL
    ```
    
    Enregistrez et quittez (`Ctrl+X`, `Y`, `Entrée` pour nano).
    

* * *

## **6\. Installation et Configuration de GRUB (Chargeur de Démarrage)**

1.  **Installer les paquets nécessaires pour GRUB UEFI :**
    
    ```bash
    pacman -S grub efibootmgr dosfstools os-prober mtools ntfs-3g
    ```
    
    - `grub` : le chargeur de démarrage.
    - `efibootmgr` : nécessaire pour l'installation de GRUB en mode UEFI.
    - `dosfstools` : outils pour les systèmes de fichiers FAT (nécessaire pour la partition EFI).
    - `os-prober` : essentiel pour détecter Windows 10.
    - `mtools` : outils pour les systèmes de fichiers MS-DOS (parfois utile).
    - `ntfs-3g` : pour lire/écrire sur les partitions NTFS de Windows.
2.  **Installer GRUB sur la partition EFI :**
    
    ```bash
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub_uefi
    ```
    
    - `--efi-directory=/boot/efi` : pointe vers la partition EFI de Windows que nous avons montée.
    - `--bootloader-id=grub_uefi` : crée une entrée nommée "grub_uefi" dans le menu de démarrage UEFI.
3.  **Activer la détection de Windows 10 par GRUB :**
    
    ```bash
    nano /etc/default/grub
    ```
    
    Ajoutez ou décommentez la ligne suivante :
    
    ```
    GRUB_DISABLE_OS_PROBER="false"
    ```
    
4.  **Générer le fichier de configuration de GRUB :**
    
    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
    
    - Cette commande va scanner les systèmes d'exploitation (y compris Windows grâce à `os-prober`) et créer le menu de démarrage. Vous devriez voir des lignes indiquant la détection de Windows.

* * *

## **7\. Installation de l'Environnement Graphique (GNOME) et du Réseau**

1.  **Installer DHCP client (pour le réseau après redémarrage) :**
    
    ```bash
    pacman -S dhcpcd
    systemctl enable dhcpcd # Active le client DHCP au démarrage
    ```
    
2.  **Installer Xorg (serveur graphique optionnel) :**
    
    ```bash
    pacman -S xorg
    ```
    
3.  **Installer l'environnement de bureau GNOME :**
    
    ```bash
    pacman -S gnome
    ```
    
4.  **Activer les services nécessaires :**
    
    ```bash
    systemctl enable gdm # Active le gestionnaire d'affichage GNOME
    systemctl enable NetworkManager # Active le service de gestion de réseau
    ```
    

* * *

## **8\. Finalisation et Redémarrage**

1.  **Quitter le chroot :**
    
    ```bash
    exit
    ```
    
2.  **Démonter toutes les partitions :**
    
    ```bash
    umount -a
    ```
    
    (Ignorez les messages d'erreur si certaines partitions ne peuvent pas être démontées, cela arrive si elles sont toujours utilisées par le système live).
3.  **Redémarrer le système :**
    
    ```bash
    telinit 6
    ```
    
    Ou simplement `reboot`.

Au redémarrage, vous devriez voir le menu de GRUB avec des options pour démarrer Arch Linux et Windows Boot Manager. Sélectionnez Arch Linux pour démarrer votre nouvelle installation.

* * *

## **9\. Après le premier redémarrage (si besoin)**

Si vous perdez la connexion Internet après le redémarrage car le client DHCP n'est pas en cours d'exécution :

```bash
sudo systemctl start dhcpcd
sudo systemctl enable dhcpcd # Active le client DHCP pour les futurs démarrages
ip a # Vérifiez votre configuration IP
ping -c 2 google.com # Vérifiez la connectivité
```
