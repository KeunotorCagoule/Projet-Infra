# Projet - Infra

> Projet réalisé par ROULLAND Roxanne, LAROUMANIE Gabriel et VASSEUR Alexis

Pour ce projet, nous avons décider de monter une plateforme de streaming audio grâce au service nommé Funkwhale.

## Sommaire
- [Setup users + groups](#setup-users--groups)
- [Sécurisation basique du VPS](#sécurisation-basique-du-vps)
    - [Mise à jour du système](#mise-à-jour-du-système)
    - [SSH](#ssh)
        - [Modifications du port SSH](#modifications-du-port-ssh)
        - [Désactivation de l’accès au serveur via l’utilisateur root](#désactivation-de-l'accès-au-serveur-via-l'utilisateur-root)
    - [Installation + Config de Fail2ban](#installation--config-de-fail2ban)
    - [Firewall](#firewall)
- [Docker](#docker)
  - [Setup du repo](#setup-du-repo)
  - [Install de Docker Engine](#install-de-docker-engine)
  - [Install de Docker Compose](#install-de-docker-compose)
- [Hostname](#hostname)
- [Certificat SSL](#certificat-ssl)
- [Funkwhale](#funkwhale)

## Setup users + groups

Après avoir acheté la machine virtuelle, nous commençons par créer un utilisateur chacun ainsi qu'un groupe pour ceux-ci afin d'être dans les *sudoers*.

On commence donc par créer le groupe et vérifions qu'il est bel et bien créer dans le fichier contenant tout les groupes existant sur la machine :
```sh
# Create a group
$ sudo groupadd admin
# Check if it's ok
$ grep admin /etc/group
admin:x:1001:
```

Ensuite nous créons donc les 3 utilisateur, nous vérifions qu'ils soient bien présent dans le fichiers contenant les utilisateurs.

Puis nous les ajoutons dans le groupe précédement créé. Et nous incluons ces utilisateurs dans un second group nommé 'sudo' pour pouvoir utiliser la commande sudo.

```sh
# Create user with his home directory (-m), a password (-p), and don't create a group with the same name as the user (-N)
$ sudo useradd -m -p <password> -N <username>
# Check if it's ok
$ grep <username> /etc/passwd
<username>:x:1003:100::/home/<username>:/bin/sh

# Set the group "admin" as the primary group of the user
$ sudo usermod -g admin <username>

# Add user in the group sudo to use sudo commands
$ usermod -aG sudo <username>
# Check if it's ok
$ grep sudo /etc/group
sudo:x:27:debian,gabriel,alexis,roxanne

```


## Sécurisation basique du VPS
### Mise à jour du système

On commence la sécurisation de la machine en la mettant à jour :
```sh
$ sudo apt-get update
$ sudo apt-get upgrade
```

### SSH
#### Modifications du port SSH
Afin d'éviter un maximum d'attaques sur le SSH, nous changeont sont port :
```sh
# Show the port of the SSH server
$ grep Port /etc/ssh/sshd_config
Port 22

# Edit the SSH config file
$ nano /etc/ssh/sshd_config

# Show the new configured port
$ grep Port /etc/ssh/sshd_config
Port 51356

# Apply the new configuration
$ systemctl restart sshd
```

Pour se connecter nous devrons donc faire :
```sh
$ ssh username@ip -p 51356
```

#### Désactivation de l’accès au serveur via l’utilisateur root
Toujours sur le SSH, on désactive l'accès à distance de l'utilisateur 'root', qui pour rappel, possède tout les droits sur la machine.

```sh
# Edit the SSH config file
$ sudo nano /etc/ssh/sshd_config

# Show the updated line
$ grep PermitRootLogin /etc/ssh/sshd_config
PermitRootLogin no

# Apply the changes
$systemctl restart sshd
```

### Installation + Config de Fail2ban
Ensuite nous installons et configurons le service [Fail2ban](https://doc.ubuntu-fr.org/fail2ban).

> Fail2ban est une application qui analyse les logs de divers services (SSH, Apache, FTP…) en cherchant des correspondances entre des motifs définis dans ses filtres et les entrées des logs.<br />
> Lorsqu'une correspondance est trouvée une ou plusieurs actions sont exécutées. <br/>
> Typiquement, fail2ban cherche des tentatives répétées de connexions infructueuses dans les fichiers journaux et procède à un bannissement en ajoutant une règle au pare-feu iptables ou nftables pour bannir l'adresse IP de la source.

```sh
# Install fail2ban
$ sudo apt-get install fail2ban

# Reload the default configuration to see if the installation is successful
$ sudo fail2ban-client reload
OK

# Show the default jails
$ sudo fail2ban-client status
Status
|- Number of jail:      3
`- Jail list:   recidive, sshd, sshd-ddos
```

### Firewall
Maintenant, nous mettons en place le firewall.
<br />
Pour notre projet, nous avons besoin d'autoriser les ports :

- 443 => Port par défaut pour un serveur web sécurisé (= https) grâce à un certificat SSL.
- 51356 => Port custom pour la connexion au serveur SSH

Dans notre cas, le service **firewalld** est déjà installé.

```sh
# Install firewall-cmd 
$ sudo apt install firewalld

# Add ports to the firewall
$ sudo firewall-cmd --add-port=80/tcp --permanent
$ sudo firewall-cmd --add-port=51356/tcp --permanent

# Reload the config with these new ports
$ sudo firewall-cmd --reload

# Start the firewall service
$ sudo systemctl start firewalld

# Set automatic start of the firewall at launch
$ sudo systemctl enable firewalld
```

## Docker

Docker est :


### Setup du repo

<br />
On commence par installer les packages nécessaires à `apt` pour utiliser un repo par HTTPS
```sh
$ sudo apt-get install \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```

On ajoute la clé GPG officielle de Docker

```sh
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

On setup le repo **stable**

```sh
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install de Docker Engine

On installe maintenant Docker Engine

```sh
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

On vérifie ensuite son installation
> Cette commande télécharge une image test et l'éxécute dans un container. Quand il tourne, il affiche un message et s'éteint.

```sh
$ sudo docker run hello-world
```

### Install de Docker Compose

Pour installer Funkwhale, il est nécessaire d'installer également Docker Compose qui est un outils permettant de définir et de partager des applications multi-containers.

On commence donc par le télécharger
```sh
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

On ajoute les permissions d'exécution sur le binaire:

```sh
sudo chmod +x /usr/local/bin/docker-compose
```

Et enfin on vérifie l'installation:

```sh
$ docker-compose --version
docker-compose version 1.29.2, build 1110ad01
```

## Hostname
Maintenant que le site est en place, on change le hostname de la machine afin d'accéder au site par le hostname au lieu de l'adresse IP de la machine.
<br />
Le hostname que nous avons pris gratuitement sur [No-IP](https://noip.com) est [https://husic.servebeer.com/](https://husic.servebeer.com/).

```sh
$ hostname husic.servebeer.com

$ sudo nano /etc/hostname

$ cat /etc/hostname
husic.servebeer.com
```

Puis lorsqu'on redémarre la machine on vérifie son application en faisant:

```sh
host husic.servebeer.com
```

## Certificat SSL

<<<<<<<<<<<<<<<<<>>>>>>>>>>>>>>>>>

## Funkwhale

Pour installer et configurer Funkwhale, nous commençons par exporter la version que l'on souhaite utiliser:

```sh
$ export FUNKWHALE_VERSION="1.2.2
```

On télécharge les config de docker-compose par défaut:

```sh
$ mkdir /srv/funkwhale
$ cd /srv/funkwhale
$ mkdir nginx

$ curl -L -o nginx/funkwhale.template "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/${FUNKWHALE_VERSION}/deploy/docker.nginx.template"

$ curl -L -o nginx/funkwhale_proxy.conf "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/${FUNKWHALE_VERSION}/deploy/docker.funkwhale_proxy.conf"

$ curl -L -o docker-compose.yml "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/${FUNKWHALE_VERSION}/deploy/docker-compose.yml"
```

On créer un fichier d'environnement `.env`

```sh
$ curl -L -o .env "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/${FUNKWHALE_VERSION}/deploy/env.prod.sample"

$ sed -i "s/FUNKWHALE_VERSION=latest/FUNKWHALE_VERSION=$FUNKWHALE_VERSION/" .env

$ chmod 600 .env

$ sudo nano .env
```

Ensuite, on récupère les images requises:

```sh
$ docker-compose pull
```

On lance le container de la BDD et la migration initiale

```sh
$ docker-compose up -d postgres

$ docker-compose run --rm api python manage.py migrate
```

Puis nous créons un utilisateur funkwhale admin:

```sh
$ docker-comose run --rm api python manage.py createsuperuser
```

Puis nous lançons funkwhale dans son entièretée:

```sh
$ docker-comose up -d
```