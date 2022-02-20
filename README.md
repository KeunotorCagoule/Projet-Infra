# Projet - Infra

> Projet réalisé par ROULLAND Roxanne, LAROUMANIE Gabriel et VASSEUR Alexis

Pour ce projet, nous avons décidé de monter une plateforme de streaming audio grâce au service nommé Funkwhale.

## Sommaire

- [Prérequis](#prérequis)
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
- [Certificat TLS](#certificat-tls)
- [Funkwhale](#funkwhale)
- [Reverse Proxy](#reverse-proxy)
- [Monitoring](#monitoring)
- [Backup](#backup)

## Prérequis

Pour ce projet, nous avons utilisé un VPS (par soucis de budget) qui utilise Debian 11 avec 2Go de RAM, et 1 vCore.

## Setup users + groups

Après avoir acheté la machine virtuelle, nous commençons par **nous** créer un utilisateur chacun ainsi qu'un groupe afin d'être dans les *sudoers*.

On commence donc par créer le groupe et vérifions qu'il est bel et bien créé dans le fichier contenant tous les groupes existants sur la machine :

```sh
# Create a group
$ sudo groupadd admin
# Check if it's ok
$ grep admin /etc/group
admin:x:1001:
```

Ensuite nous créons donc les 3 utilisateurs, nous vérifions qu'ils sont bien présents dans le fichier contenant les utilisateurs.

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
sudo apt-get update
sudo apt-get upgrade
```

### SSH

#### Modifications du port SSH

Afin d'éviter un maximum d'attaques sur le SSH, nous **changeons** **son** port :

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
ssh username@ip -p 51356
```

#### Désactivation de l’accès au serveur via l’utilisateur root

Toujours sur le SSH, on désactive l'accès à distance de l'utilisateur 'root', qui pour rappel, possède tous les droits sur la machine.

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

> Fail2ban est une application qui analyse les logs de divers services (SSH, Apache, FTP…) en cherchant des correspondances entre des motifs définis dans ses filtres et les entrées des logs
>
> Lorsqu'une correspondance est trouvée une ou plusieurs actions sont exécutées.
>
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

Pour notre projet, nous avons besoin d'autoriser les ports :

- 443 => Port par défaut pour un serveur web sécurisé (= https) grâce à un certificat TLS.
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

On commence par installer les packages nécessaires à `apt` pour utiliser un repo par HTTPS.

```sh
$ sudo apt-get install \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```

On ajoute la clé GPG officielle de Docker.

```sh
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

On setup le repo **stable**

```sh
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install de Docker Engine

On installe maintenant Docker Engine.

```sh
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

On vérifie ensuite son installation.
> Cette commande télécharge une image test et l'éxécute dans un container. Quand il tourne, il affiche un message et s'éteint.

```sh
sudo docker run hello-world
```

### Install de Docker Compose

Pour installer Funkwhale, il est nécessaire d'installer également Docker Compose qui est un outil permettant de définir et de partager des applications multi-containers.

On commence donc par le télécharger.

```sh
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
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

## Funkwhale

Pour installer et configurer Funkwhale, nous commençons par exporter la version que l'on souhaite utiliser:

```sh
export FUNKWHALE_VERSION="1.2.2"
```

On télécharge les config de docker-compose par défaut:

```sh
mkdir /srv/funkwhale
cd /srv/funkwhale
mkdir nginx

curl -L -o nginx/funkwhale.template "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/${FUNKWHALE_VERSION}/deploy/docker.nginx.template"

curl -L -o nginx/funkwhale_proxy.conf "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/${FUNKWHALE_VERSION}/deploy/docker.funkwhale_proxy.conf"

curl -L -o docker-compose.yml "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/${FUNKWHALE_VERSION}/deploy/docker-compose.yml"
```

On crée un fichier d'environnement `.env`.

```sh
curl -L -o .env "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/${FUNKWHALE_VERSION}/deploy/env.prod.sample"

sed -i "s/FUNKWHALE_VERSION=latest/FUNKWHALE_VERSION=$FUNKWHALE_VERSION/" .env

chmod 600 .env

sudo nano .env
```

Ensuite, on récupère les images requises:

```sh
docker-compose pull
```

On lance le container de la BDD et la migration initiale.

```sh
docker-compose up -d postgres

docker-compose run --rm api python manage.py migrate
```

Puis nous créons un utilisateur funkwhale admin:

```sh
docker-comose run --rm api python manage.py createsuperuser
```

Puis nous lançons funkwhale dans son entièreté:

```sh
docker-comose up -d
```

## Reverse Proxy

DEF

Pour installer le Reverse Proxy, nous exécutons ces commandes :

On télécharge les fichiers requis.

```sh
curl -L -o /etc/nginx/funkwhale_proxy.conf "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/|version|/deploy/funkwhale_proxy.conf"

curl -L -o /etc/nginx/sites-available/funkwhale.template "https://dev.funkwhale.audio/funkwhale/funkwhale/raw/|version|/deploy/docker.proxy.template"
```

On crée une config nginx finale en utilisant le template basé sur le fichier d'environnement

```sh
set -a && source /srv/funkwhale/.env && set +a
envsubst "`env | awk -F = '{printf \" $%s\", $$1}'`" \
    < /etc/nginx/sites-available/funkwhale.template \
    > /etc/nginx/sites-available/funkwhale.conf
```

Puis finalement, on active la config finale:

```sh
ln -s /etc/nginx/sites-available/funkwhale.conf /etc/nginx/sites-enabled/
```

> A partir d'ici, l'application est mise en place et fonctionnelle.
> Pour la voir, rendez-vous à l'adresse IP de votre serveur, ou si vous avez pris un nom de domaine, utilisez celle-ci.

## Monitoring

Pour commencer, nous avons besoin de créer un nouvel utilisateur qui sera appelé `prometheus`:

```sh
sudo adduser prometheus

Adding user 'prometheus' ...

Adding new group 'prometheus' (1002) ...

Adding new user 'prometheus' (1004) with group 'prometheus' ...

Creating home directory '/home/prometheus' ...

Copying files from '/etc/skel' ...

New password: 

Retype new password: 

passwd: password updated successfully

Changing the user information for prometheus

Enter the new value, or press ENTER for the default

    Full Name []: 

    Room Number []: 

    Work Phone []: 

    Home Phone []: 

    Other []: 

Is the information correct? [Y/n] Y
```

Puis on l'ajoute au groupe docker pour qu'il puisse créer et lancer les containers prometheus.

```sh
sudo adduser prometheus docker

Adding user 'prometheus' to group 'docker' ...

Adding user prometheus to group docker

Done.
```

On se connecte à ce compte pour réaliser la config sous celui-ci.

```sh
su prometheus
```

On crée un dossier de monitoring et un fichier de config dans celui-ci:

```sh
mkdir monitoring
cd monitoring
nano docker-compose.yml
cat docker-compose.yml 

version: '3'

services:

  prometheus:

    image: prom/prometheus:latest

    user: 1004:1004

    container_name: prometheus

    volumes:

      - ./prometheus.yml:/etc/prometheus/prometheus.yml

      - ./prometheus_db:/var/lib/prometheus

      - ./prometheus_db:/prometheus

      - ./prometheus_db:/etc/prometheus

      - ./alert.rules:/etc/prometheus/alert.rules

    command:

      - '--config.file=/etc/prometheus/prometheus.yml'

      - '--web.route-prefix=/'

      - '--storage.tsdb.retention.time=200h'

      - '--web.enable-lifecycle'

    restart: unless-stopped

    ports:

      - 127.0.0.1:9090:9090
```

Puis nous créons le fichier de config prometheus.

On notera qu'au fur et à mesure de la conf on continuera à modifier le MÊME fichier docker-compose.

On crée le dossier du container prometheus, on le lance et vérifions qu'il soit bien lancé.

```sh
mkdir prometheus_db
docker-compose up -d

Recreating prometheus ... done

docker ps

CONTAINER ID   IMAGE                       COMMAND                  CREATED         STATUS         PORTS                                         NAMES

d5161c50b263   prom/prometheus:latest      "/bin/prometheus --c…"   4 seconds ago   Up 3 seconds   0.0.0.0:9090->9090/tcp, :::9090->9090/tcp     prometheus

123213f66681   nginx                       "/docker-entrypoint.…"   7 days ago      Up 7 days      127.0.0.1:5000->80/tcp                        funkwhale_nginx_1

ea0d0b4a4e36   funkwhale/funkwhale:1.2.2   "./compose/django/en…"   7 days ago      Up 7 days                                                    funkwhale_celeryworker_1

b8f1f43d7014   funkwhale/funkwhale:1.2.2   "./compose/django/en…"   7 days ago      Up 7 days      0.0.0.0:49156->5000/tcp, :::49156->5000/tcp   funkwhale_api_1

6eb3577bbec9   funkwhale/funkwhale:1.2.2   "./compose/django/en…"   7 days ago      Up 7 days                                                    funkwhale_celerybeat_1

5916daa2d839   redis:5                     "docker-entrypoint.s…"   7 days ago      Up 7 days      6379/tcp                                      funkwhale_redis_1

220e9671d2ec   postgres:11                 "docker-entrypoint.s…"   7 days ago      Up 7 days      5432/tcp                                      funkwhale_postgres_1
```

On ajoute ensuite dans la config du container ainsi que dans la conf prometheus :

- Node Exporter : Récupération et exports des informations de la machine hôte
- CAdvisor : Outils permettant de récupérer des informations sur l'installation docker qui tourne dessus
- Grafana : outils permettant la génération de dashboards orienté data

```sh
cat docker-compose.yml
version: '3'
services:
  prometheus:
    image: prom/prometheus:latest
    networks:
      - default
    user: "1004:1004"
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus_db:/var/lib/prometheus
      - ./prometheus_db:/prometheus
      - ./prometheus_db:/etc/prometheus
      - ./alert.rules:/etc/prometheus/alert.rules
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.route-prefix=/'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    restart: unless-stopped
    ports:
      - 127.0.0.1:9090:9090
  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    networks:
      - default
    ports:
      - 127.0.0.1:9100:9100
  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    networks:
      - default
    container_name: cadvisor
    ports:
      - 127.0.0.1:8080:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
  grafana:
    image: grafana/grafana
    networks:
      - default
    ports:
      - 127.0.0.1:3000:3000
    restart: unless-stopped
    volumes:
      - grafana-data:/var/lib/grafana


volumes:
  grafana-data:

networks:
  default:`
```

```sh
nano prometheus.yml
cat prometheus.yml

global:
  scrape_interval: 5s
  external_labels:
    monitor: 'prakash-monitor'
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090'] ## IP Address of the localhost
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
  - job_name: 'cAdvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

Puis nous relançons.

```sh
docker-compose pull
docker-compose down
docker-compose up -d
```

On vérifie que tout va bien.

```sh
docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED        STATUS                  PORTS                                         NAMES
75b14c322f96   prom/prometheus:latest      "/bin/prometheus --c…"   33 hours ago   Up 32 hours             127.0.0.1:9090->9090/tcp                      prometheus
97535a5e1b5b   grafana/grafana             "/run.sh"                33 hours ago   Up 32 hours             127.0.0.1:3000->3000/tcp                      monitoring_grafana_1
11722cd24d20   gcr.io/cadvisor/cadvisor    "/usr/bin/cadvisor -…"   33 hours ago   Up 32 hours (healthy)   127.0.0.1:8080->8080/tcp                      cadvisor
dc8863fc3534   prom/node-exporter          "/bin/node_exporter"     33 hours ago   Up 32 hours             127.0.0.1:9100->9100/tcp                      node-exporter
123213f66681   nginx                       "/docker-entrypoint.…"   8 days ago     Up 8 days
     127.0.0.1:5000->80/tcp                        funkwhale_nginx_1
ea0d0b4a4e36   funkwhale/funkwhale:1.2.2   "./compose/django/en…"   8 days ago     Up 8 days
                                                   funkwhale_celeryworker_1
b8f1f43d7014   funkwhale/funkwhale:1.2.2   "./compose/django/en…"   8 days ago     Up 8 days
     0.0.0.0:49156->5000/tcp, :::49156->5000/tcp   funkwhale_api_1
6eb3577bbec9   funkwhale/funkwhale:1.2.2   "./compose/django/en…"   8 days ago     Up 8 days
                                                   funkwhale_celerybeat_1
5916daa2d839   redis:5                     "docker-entrypoint.s…"   8 days ago     Up 8 days
     6379/tcp                                      funkwhale_redis_1
220e9671d2ec   postgres:11                 "docker-entrypoint.s…"   8 days ago     Up 8 days
     5432/tcp
```

## Backup

Pour commencer la configuration d'un script de backup, nous devons créer un utilisateur nommé `autobackup`.

Les fichiers de backups seront placés dans `/home/autobackup/backupsdatabase/postgresql/`.

```sh
sudo useradd autobackup
```

Ensuite dans son dossier home, nous créons 2 fichiers, un de config et le script.

La config `pg_backup.config`:

```sh
$ cat pg_backup.config
##############################
## POSTGRESQL BACKUP CONFIG ##
##############################

# Optional system user to run backups as.  If the user the script is running as doesn't match this
# the script terminates.  Leave blank to skip check.
BACKUP_USER=gabriel

# Optional hostname to adhere to pg_hba policies.  Will default to "localhost" if none specified.
HOSTNAME=

# Optional username to connect to database as.  Will default to "postgres" if none specified.
USERNAME=

# This dir will be created if it doesn't exist.  This must be writable by the user the script is
# running as.
BACKUP_DIR=/home/autobackup/backupsdatabase/postgresql/

# List of strings to match against in database name, separated by space or comma, for which we only
# wish to keep a backup of the schema, not the data. Any database names which contain any of these
# values will be considered candidates. (e.g. "system_log" will match "dev_system_log_2010-01")
SCHEMA_ONLY_LIST=""

# Will produce a custom-format backup if set to "yes"
ENABLE_CUSTOM_BACKUPS=yes

# Will produce a gzipped plain-format backup if set to "yes"
ENABLE_PLAIN_BACKUPS=yes

# Will produce gzipped sql file containing the cluster globals, like users and passwords, if set to "yes"
ENABLE_GLOBALS_BACKUPS=yes


#### SETTINGS FOR ROTATED BACKUPS ####

# Which day to take the weekly backup from (1-7 = Monday-Sunday)
DAY_OF_WEEK_TO_KEEP=1-7

# Number of days to keep daily backups
DAYS_TO_KEEP=7

# How many weeks to keep weekly backups
WEEKS_TO_KEEP=4

######################################
```

Le script `pg_backup_rotated.sh`:

```sh
cat pg_backup_rotated.sh
#!/bin/bash

###########################
####### LOAD CONFIG #######
###########################

while [ $# -gt 0 ]; do
        case $1 in
                -c)
                        CONFIG_FILE_PATH="$2"
                        shift 2
                        ;;
                *)
                        ${ECHO} "Unknown Option \"$1\"" 1>&2
                        exit 2
                        ;;
        esac
done

if [ -z $CONFIG_FILE_PATH ] ; then
        SCRIPTPATH=$(cd ${0%/*} && pwd -P)
        CONFIG_FILE_PATH="${SCRIPTPATH}/pg_backup.config"
fi

if [ ! -r ${CONFIG_FILE_PATH} ] ; then
        echo "Could not load config file from ${CONFIG_FILE_PATH}" 1>&2
        exit 1
fi

source "${CONFIG_FILE_PATH}"

###########################
#### PRE-BACKUP CHECKS ####
###########################

# Make sure we're running as the required backup user
if [ "$BACKUP_USER" != "" -a "$(id -un)" != "$BACKUP_USER" ] ; then
        echo "This script must be run as $BACKUP_USER. Exiting." 1>&2
        exit 1
fi


###########################
### INITIALISE DEFAULTS ###
###########################

if [ ! $HOSTNAME ]; then
        HOSTNAME="localhost"
fi;

if [ ! $USERNAME ]; then
        USERNAME="postgres"
fi;


###########################
#### START THE BACKUPS ####
###########################

function perform_backups()
{
        SUFFIX=$1
        FINAL_BACKUP_DIR=$BACKUP_DIR"`date +\%Y-\%m-\%d`$SUFFIX/"

        echo "Making backup directory in $FINAL_BACKUP_DIR"

        if ! mkdir -p $FINAL_BACKUP_DIR; then
                echo "Cannot create backup directory in $FINAL_BACKUP_DIR. Go and fix it!" 1>&2
                exit 1;
        fi;

        #######################
        ### GLOBALS BACKUPS ###
        #######################

        echo -e "\n\nPerforming globals backup"
        echo -e "--------------------------------------------\n"

        if [ $ENABLE_GLOBALS_BACKUPS = "yes" ]
        then
                    echo "Globals backup"

                    set -o pipefail
                    if ! pg_dumpall -g -h "$HOSTNAME" -U "$USERNAME" | gzip > $FINAL_BACKUP_DIR"globals".sql.gz.in_progress; then
                            echo "[!!ERROR!!] Failed to produce globals backup" 1>&2
                    else
                            mv $FINAL_BACKUP_DIR"globals".sql.gz.in_progress $FINAL_BACKUP_DIR"globals".sql.gz
                    fi
                    set +o pipefail
        else
                echo "None"
        fi


        ###########################
        ### SCHEMA-ONLY BACKUPS ###
        ###########################

        for SCHEMA_ONLY_DB in ${SCHEMA_ONLY_LIST//,/ }
        do
                SCHEMA_ONLY_CLAUSE="$SCHEMA_ONLY_CLAUSE or datname ~ '$SCHEMA_ONLY_DB'"
        done

        SCHEMA_ONLY_QUERY="select datname from pg_database where false $SCHEMA_ONLY_CLAUSE order by datname;"

        echo -e "\n\nPerforming schema-only backups"
        echo -e "--------------------------------------------\n"

        SCHEMA_ONLY_DB_LIST=`psql -h "$HOSTNAME" -U "$USERNAME" -At -c "$SCHEMA_ONLY_QUERY" postgres`

        echo -e "The following databases were matched for schema-only backup:\n${SCHEMA_ONLY_DB_LIST}\n"

        for DATABASE in $SCHEMA_ONLY_DB_LIST
        do
                echo "Schema-only backup of $DATABASE"
                set -o pipefail
                if ! pg_dump -Fp -s -h "$HOSTNAME" -U "$USERNAME" "$DATABASE" | gzip > $FINAL_BACKUP_DIR"$DATABASE"_SCHEMA.sql.gz.in_progress; then
                        echo "[!!ERROR!!] Failed to backup database schema of $DATABASE" 1>&2
                else
                        mv $FINAL_BACKUP_DIR"$DATABASE"_SCHEMA.sql.gz.in_progress $FINAL_BACKUP_DIR"$DATABASE"_SCHEMA.sql.gz
                fi
                set +o pipefail
        done


        ###########################
        ###### FULL BACKUPS #######
        ###########################

        for SCHEMA_ONLY_DB in ${SCHEMA_ONLY_LIST//,/ }
        do
                EXCLUDE_SCHEMA_ONLY_CLAUSE="$EXCLUDE_SCHEMA_ONLY_CLAUSE and datname !~ '$SCHEMA_ONLY_DB'"
        done

        FULL_BACKUP_QUERY="select datname from pg_database where not datistemplate and datallowconn $EXCLUDE_SCHEMA_ONLY_CLAUSE order by datname;"

        echo -e "\n\nPerforming full backups"
        echo -e "--------------------------------------------\n"

        for DATABASE in `psql -h "$HOSTNAME" -U "$USERNAME" -At -c "$FULL_BACKUP_QUERY" postgres`
        do
                if [ $ENABLE_PLAIN_BACKUPS = "yes" ]
                then
                        echo "Plain backup of $DATABASE"

                        set -o pipefail
                        if ! pg_dump -Fp -h "$HOSTNAME" -U "$USERNAME" "$DATABASE" | gzip > $FINAL_BACKUP_DIR"$DATABASE".sql.gz.in_progress; then
                                echo "[!!ERROR!!] Failed to produce plain backup database $DATABASE" 1>&2
                        else
                                mv $FINAL_BACKUP_DIR"$DATABASE".sql.gz.in_progress $FINAL_BACKUP_DIR"$DATABASE".sql.gz
                        fi
                        set +o pipefail

                fi

                if [ $ENABLE_CUSTOM_BACKUPS = "yes" ]
                then
                        echo "Custom backup of $DATABASE"

                        if ! pg_dump -Fc -h "$HOSTNAME" -U "$USERNAME" "$DATABASE" -f $FINAL_BACKUP_DIR"$DATABASE".custom.in_progress; then
                                echo "[!!ERROR!!] Failed to produce custom backup database $DATABASE"
                        else
                                mv $FINAL_BACKUP_DIR"$DATABASE".custom.in_progress $FINAL_BACKUP_DIR"$DATABASE".custom
                        fi
                fi

        done

        echo -e "\nAll database backups complete!"
}

# MONTHLY BACKUPS

DAY_OF_MONTH=`date +%d`

if [ $DAY_OF_MONTH -eq 1 ];
then
        # Delete all expired monthly directories
        find $BACKUP_DIR -maxdepth 1 -name "*-monthly" -exec rm -rf '{}' ';'

        perform_backups "-monthly"

        exit 0;
fi

# WEEKLY BACKUPS

DAY_OF_WEEK=`date +%u` #1-7 (Monday-Sunday)
EXPIRED_DAYS=`expr $((($WEEKS_TO_KEEP * 7) + 1))`

if [ $DAY_OF_WEEK = $DAY_OF_WEEK_TO_KEEP ];
then
        # Delete all expired weekly directories
        find $BACKUP_DIR -maxdepth 1 -mtime +$EXPIRED_DAYS -name "*-weekly" -exec rm -rf '{}' ';'

        perform_backups "-weekly"

        exit 0;
fi

# DAILY BACKUPS

# Delete daily backups 7 days old or more
find $BACKUP_DIR -maxdepth 1 -mtime +$DAYS_TO_KEEP -name "*-daily" -exec rm -rf '{}' ';'

perform_backups "-daily"
```

Puis nous créons un service qui permettera de laisser tourner le script:

Dans `/etc/systemd/system/`, nous créons `backup.service`:

```sh
cat /etc/systemd/system/backup.service
[Unit]
Description=Backup Script

[Service]
ExecStart=/home/autobackup/pg_backup_rotated.sh
Group=admin


[Install]
WantedBy=multi-user.target
```

Puis nous lançons le service et vérifions qu'il tourne bien.

```sh
sudo systemctl start backup.service

sudo systemctl enable backup.service

systemctl status backup.service
● backup.service - Backup Script
     Loaded: loaded (/etc/systemd/system/backup.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-02-15 15:08:47 UTC; 5 days ago
    Process: 630640 ExecStart=/home/autobackup/pg_backup_rotated.sh 
   Main PID: 630640 (backup)
        CPU: 1ms
...
```
