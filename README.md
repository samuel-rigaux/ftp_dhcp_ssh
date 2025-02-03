
# RAPPORT

# 1. Installation de Debian sans interface graphique

J'ai commencé par créer 2 machines virtuelles (VM) avec, pour chacune d'elles, 8 Go de RAM et 2 cœurs de processeur. Ensuite, j'ai lancé l'installation de Debian en mode classique, en choisissant de ne pas installer GNOME ou MATE.

# 2. Mise à jour des systèmes :

Pour mettre à jour les systèmes, j'ai d'abord permis à mon utilisateur "samuel" d'être dans les sudoers avec les commandes suivantes :

    su -
    
    usermod -aG sudo Samuel
    
    exit

Ensuite, j'ai exécuté `sudo apt update` et `sudo apt upgrade` pour effectuer les mises à jour.

# 3. Configuration du serveur DHCP :

J'ai tout d'abord mis à jour le système avec :

    sudo apt update && apt upgrade -y

Ensuite, j'ai installé le paquet isc-dhcp-server avec la commande :

    sudo apt install isc-dhcp-server -y

J'ai édité le fichier de configuration du DHCP avec :

    sudo nano /etc/dhcp/dhcpd.conf

Et j'ai ajouté les lignes suivantes :

    subnet 172.16.0.0 netmask 255.255.0.0 {
    
    range 172.16.1.2 172.16.2.253;
    
    option routers 172.16.0.1;
    
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    
    }

Ensuite, j'ai spécifié l'interface réseau DHCP avec :

    nano /etc/default/isc-dhcp-server

Et dans la ligne `INTERFACESv4=""`, j'ai ajouté `ens33`.

Après cela, j'ai configuré l'interface réseau avec :

    nano /etc/network/interfaces

Puis j'ai ajouté :

    auto ens33
    
    iface ens33 inet static
    
    address 172.16.0.2
    
    netmask 255.255.0.0
    
    gateway 192.168.127.55 # Gateway que j'ai trouvé en faisant ip route sur Debian

Ensuite, j'ai appliqué les changements réseau en redémarrant la carte réseau :

    ifdown ens33 && ifup ens33

Finalement, j'ai lancé le service DHCP avec :

    systemctl start isc-dhcp-server
    
    systemctl status isc-dhcp-server

# 4. Installation du serveur FTP et SSH :

## SSH :

Tout d'abord, j'ai lancé ma VM2 et j'ai installé Debian en incluant par défaut le serveur SSH. Ensuite, j'ai effectué une mise à jour avec :

    apt update && apt upgrade -y

Puis, j'ai exécuté les commandes suivantes en tant que superutilisateur (su -) :

    systemctl start ssh
    
    systemctl enable ssh

Après cela, je suis allé dans PowerShell sur mon Windows hôte pour accéder à ma VM via SSH. J'ai donc exécuté :

    ssh samuel@172.16.0.102

En renseignant mon mot de passe, la connexion a fonctionné. Depuis mon terminal qui contrôle la VM, j'ai commencé à préparer les connexions FTP.

## FTP :

J'ai installé le serveur FTP avec :

    apt install proftpd -y

Ensuite, je l'ai lancé avec :

    systemctl start proftpd
    
    systemctl enable proftpd

J'ai suivi le tutoriel de IT-Connect pour configurer le serveur FTP. J'ai commencé par créer un fichier de configuration pour le FTP :

    nano /etc/proftpd/conf.d/ftp-perso.conf

Puis je l'ai renseigné avec les données suivantes :

    #Nom du serveur (identique à celui défini dans /etc/hosts)
    
    ServerName "debianHTPSSH"
    
    #Message de connexion
    
    DisplayLogin "Bienvenue sur le FTP de Samuel !"
    
    #Désactiver IPv6
    
    UseIPv6 off
    
    #Chaque utilisateur accède seulement à son home (pour les membres du groupe ftp2100)
    
    DefaultRoot ~ ftp2100
    
    #Port (défaut = 21)
    
    Port 21
    
    #Refuser la connexion super-utilisateur "root"
    
    RootLogin off
    
    #Nombre de clients FTP max.
    
    MaxClients 5
    
    #Autoriser la connexion seulement aux membres du groupe "ftp2100" grâce à la directive DenyGroup.
    
    #En précisant "!" on refuse tout sauf le groupe "ftp2100"
    
    <Limit LOGIN>
    
    DenyGroup !ftp2100
    
    </Limit>

Ensuite, j'ai sauvegardé et relancé le service FTP avec :

    systemctl reload proftpd

J'ai ensuite créé un nouvel utilisateur qui aura seul accès au FTP. J'ai commencé par créer le groupe auquel il appartiendra :

    addgroup ftp2100

Puis j'ai créé l'utilisateur :

    adduser laplateforme

En renseignant le mot de passe Marseille13! comme indiqué dans le sujet.

Ensuite, j'ai ajouté l'utilisateur au groupe :

    adduser laplateforme ftp2100

Cela a permis à l'utilisateur laplateforme d'accéder au FTP, car seuls les membres du groupe ftp2100 peuvent y accéder.

Pour tester la connexion, j'ai procédé de deux manières :

### 1. Depuis une autre fenêtre CMD :

J'ai exécuté :

    ftp 172.16.0.102

Puis j'ai renseigné les identifiants :

- Utilisateur : laplateforme

- Mot de passe : Marseille13!

Cela m'a permis d'accéder au FTP, mais je n'ai pas trouvé de commande fonctionnelle pour modifier les fichiers.

### 2. Via FileZilla :

J'ai téléchargé FileZilla sur mon PC hôte. Je me suis connecté avec la même adresse IP et les mêmes identifiants. Depuis FileZilla, je peux ajouter et supprimer des fichiers, qui s'affichent ensuite dans la VM.

#### SFTP :

Pour sécuriser mes accès SSH en SFTP, j'ai commencé par modifier le fichier de configuration SSH avec la commande :

    nano /etc/ssh/sshd_config

J'ai vérifié que la ligne suivante était présente :

    Subsystem sftp /usr/lib/openssh/sftp-server

Ensuite, j'ai restreint certains utilisateurs à SFTP uniquement en ajoutant ceci à la fin du fichier :

    Match Group sftpusers
    
    ChrootDirectory /home/%u
    
    ForceCommand internal-sftp
    
    AllowTcpForwarding no
    
    X11Forwarding no

Cela limite les utilisateurs du groupe sftpusers à leur répertoire personnel et leur interdit d'accéder à un shell.

J'ai terminé par redémarrer le service SSH pour appliquer les modifications :

    sudo systemctl restart sshd

J'ai voulu séparer les utilisateurs FTP et SFTP, donc j'ai créé le groupe SFTP avec :

    sudo groupadd sftpusers

Ensuite, j'ai créé un nouvel utilisateur pour utiliser le SFTP et je l'ai ajouté au groupe sftpusers avec les commandes suivantes :

    sudo adduser samuelsftp

J'ai renseigné le mot de passe root, puis j'ai exécuté :

    sudo adduser samuelsftp sftpusers

J'ai désactivé le serveur FTP pour éviter des erreurs :

    sudo systemctl stop proftpd

# 5. Installation du serveur DNS :

Pour l'installation du serveur DNS, je me suis positionné sur ma première VM, qui dispose déjà du serveur DHCP. J'ai installé Bind9 comme suit :

    sudo apt install bind9 bind9-utils bind9-doc

Ensuite, j'ai configuré le serveur DNS avec la commande suivante :

    sudo nano /etc/bind/named.conf.options

J'ai supprimé le contenu existant pour le remplacer par :

    options {
    directory "/var/cache/bind";
    recursion yes;  # Autorise la récursion pour les clients
    allow-query { 172.16.0.0/24; };  # Autorise les requêtes depuis le réseau 172.16.0.0/24
    forwarders {
    8.8.8.8;  # Utilise Google DNS comme forwarder
    8.8.4.4;
    };
    dnssec-validation auto;  # Active la validation DNSSEC
    listen-on { 172.16.0.10; };  # Écoute uniquement sur l'adresse IP du serveur
    allow-transfer { none; };  # Désactive les transferts de zone par défaut
    };

Ensuite, j'ai créé une zone DNS pour le domaine dns.ftp.com. Pour cela, j'ai modifié le fichier de configuration local :

    sudo nano /etc/bind/named.conf.local

Et j'y ai ajouté la configuration suivante :

    zone "dns.ftp.com" {
    type master;
    file "/etc/bind/db.dns.ftp.com";
    };

Après cela, j'ai créé le fichier de zone en copiant le fichier de zone de base :

    sudo cp /etc/bind/db.local /etc/bind/db.dns.ftp.com

Puis j'ai modifié le fichier :

    sudo nano /etc/bind/db.dns.ftp.com

Et j'y ai ajouté le contenu suivant :

    ; BIND data file for dns.ftp.com
    
    ;
    
    $TTL  604800
    
    @  IN  SOA  ns1.dns.ftp.com. admin.dns.ftp.com. (
    
    2023101001  ; Serial
    
    604800  ; Refresh
    
    86400  ; Retry
    
    2419200  ; Expire
    
    604800 )  ; Negative Cache TTL
    
    ;
    
    @  IN  NS  ns1.dns.ftp.com.
    
    @  IN  A  172.16.0.10  ; Adresse IP du serveur DNS
    
    ns1  IN  A  172.16.0.10  ; Adresse IP du serveur DNS

J'ai enregistré ce fichier et j'ai effectué des tests pour vérifier la syntaxe des fichiers de zone et de configuration :

    sudo named-checkconf
    
    sudo named-checkzone dns.ftp.com /etc/bind/db.dns.ftp.com

Pour appliquer les changements, j'ai redémarré le serveur DNS et j'ai activé son démarrage automatique :

    sudo systemctl restart bind9
    
    sudo systemctl enable bind9

Ensuite, j'ai configuré le client DNS sur la deuxième machine en modifiant le fichier resolv.conf :

    sudo nano /etc/resolv.conf

Et j'ai ajouté la ligne suivante :

    nameserver 172.16.0.10

Enfin, j'ai configuré ma carte réseau sur mon PC Windows pour qu'elle se connecte au serveur DNS 172.16.0.10.

Pour le test final, j'ai exécuté la commande suivante sur la première machine (celle du DNS) :

    nslookup dns.ftp.com 172.16.0.10

Avec succès.

Sur la deuxième machine, j'ai fait la même commande, qui a également fonctionné.

# 6. Test de Connexion au Serveur SFTP :

J'ai d'abord fait un ping vers dns.ftp.com comme ceci :

    ping dns.ftp.com

Ensuite j'ai fait :

    nslookup dns.ftp.com 172.16.0.10

Pour vérifier la liaison.

Puis je me suis connecté en sftp avec :

    sftp laplateforme@dns.ftp.com

On me demande le mot de passe, puis connexion avec succès.

# 7. Paramètres de Sécurité Additionnels :

On va sur la machine client, puis on modifie le fichier de config du SSH :

    nano /etc/ssh/sshd_config

Et on y ajoute les commandes suivantes :

    Port 6500
    
    AllowUsers laplateforme
    
    PermitRootLogin no



