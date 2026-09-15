# Holodeck
## Sommaire
1. [Objectifs et contexte](#1--mise-en-place-des-vm)
2. [Mise en place des VM](#2-la-mise-en-place-des-deux-vm)
3. [Serveur DHCP et DNS](#3-configuration-dns-et-dhcp)
4. [Serveur web](#4-mise-en-place-du-serveur-web)
5. [PHP](#5-mise-en-place-de-php-7-et-8)
6. [PhpMyAdmin](#6-installation-de-PhpMYAdmin)
7. [Cockpit](#7-installation-de-Cockpit)
8. [LDAP](#8-LDAP)
9.  [ProFTPd](#9-ProFTPd)
10. [VisualStudioCode](#10-VisualStudioCode) 
11. [Pare-feu UFW ](#11-Pare-feu-UFW)
12. [CertificatTLS](#12-Genere-le-certificat-avec-mkcert)


## 1.  Mise en place des vm
### 1.1 Déploiment d'une Vm serveur et d'une Vm cliente.
Contenant un serveur Web, un serveur ftp tout deux avec un certificat TLS. Ainsi que les serveurs DNS, DHCP qui auront pour domaine starfleet.lan, LDAP, MAriaDB, l'utilisation de PHP pour le serveur web avec ngnix
### 1.2 Les contraintes
- Pas de comptes sudo
- mise en place du par-feu uniquement pour les ports requis
- serveur web avec nginx et ne https
- PHP, MariaDB, et Nginx doivent être la dernière version
## 2. La mise en place des deux VM
### 2.1 Vm serveur 
- 32 Go de stockage
>
      
- 2 GO de RAM


- 2 vCPU

     
- 2 cartes réseaux une lan et une wan


- VM en CLI



### 2.2 VM cliente
- VM en GUI
- 16 Go de stckage
  
- carte réseau sur le lan 
 

##### Sur notre VM serveur ens37 est pour le LAN

>/etc/network/interfaces
```
allow-hotplug ens37 #reseau LAN
iface ens37 inet static
     address 192.168.50.1
     netmask 255.255.255.0
    
```

```bash
 systemctl restart networking
```



## 3. configuration DNS et DHCP
### 3.1 Configuration DHCP

```bash
apt install -y isc-dhcp-server
```
>`/etc/default/isc-dhcp-server` :
```bash
INTERFACESv4="ens33"
```
>`/etc/dhcp/dhcpd.conf` :
```
authoritative;

subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.100 192.168.10.200; # plage d'adresse
    option routers 192.168.10.1;
    option domain-name-servers 192.168.10.1;
    option domain-name "starfleet.lan";
}
```
### 3.2 Serveur DNS 


Installation de DNS
```bash
apt install -y bind9 bind9utils dnsutils
```


>`/etc/bind/named.conf.local :


```bash
zone "starfleet.lan" {
    type master;
    file "/etc/bind/db.starfleet.lan";
};
```

### Faire ecouter bind9 sur ens34 (carte réseau LAN)
>/etc/bind/named.conf.options

```bash
options {
    directory "/var/cache/bind";

    listen-on { 127.0.0.1; 192.168.50.1; };
    allow-query { 127.0.0.1; 192.168.50.0/24; };

    recursion yes;
    forwarders {
        8.8.8.8;
    };
};
```

Enregistrements A dans :
>/etc/bind/db.starfleet.lan 

(ns, www8, www7, php, admin, vscore) pointant vers 192.168.50.1. /etc/bind/db.starfleet.lan

```bash
$TTL    604800
@       IN      SOA     ns.starfleet.lan. admin.starfleet.lan. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@               IN      NS      ns.starfleet.lan.
ns              IN      A       192.168.2.1
www8            IN      A       192.168.2.1
www7            IN      A       192.168.2.1
php             IN      A       192.168.2.1
admin           IN      A       192.168.2.1
vscore          IN      A       192.168.2.1
ldap            IN      A       192.168.2.1

```


```bash
systemctl start bind9
systemctl enable named.service
systemctl status bind9
```

## 4. Mise en place du serveur web
### 4.1 Installation de Nginx comme serveur web.


Installation depuis le dépot officiel de nginx
```bash
apt update
apt install -y curl gnupg2 ca-certificates lsb-release debian-archive-keyring
```


Importer la clé GPG
```bash
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor | tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```

Ajout au dépôt officiel sur notre ordinateur 
```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/mainline/debian $(lsb_release -cs) nginx" | tee /etc/apt/sources.list.d/nginx.list
```
Epingler le dépôt (pas de melange avec la version Debian)
```bash
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900" | tee /etc/apt/preferences.d/99nginx
```
Installation de nginx
```bash
apt install  nginx -y
```
Verifier que la version nginx est celle de nginx.org

```bash
nginx -v
apt policy nginx #attendre un peuavant de lancer cette commande
```
Pour un demarrage automatique au lancement de du serveur 

```bash
systemctl enable nginx
```

La racine du site se trouve dans 
```bash
/var/www/html
```
Fichier de configuration global de Nginx
```bash
 /etc/nginx/nginx.conf
```

Dossier qui contient les fichiers de configuration des sites disponibles 

```bash
/etc/nginx/sites-available/
```

Dossier qui contient les fichiers de configuration des sites actifs
```bash
/etc/nginx/sites-enabled/
```

Céation des sites web (à répéter pour ww7,ww8,php,admin) 
```bash
mkdir /var/www/www7.starfleet.lan
```

```bash
chown -R www-data:www-data /var/www/www7.starfleet.lan
```

```bash
chmod 755 /var/www/www7.starfleet.lan
```

```bash
nano /var/www/www7.starfleet.lan/index.html
```

Contenu de notre page index.html
```bash
<html>
<head></head>
<body>
<h1>Bienvenue sur 7StarFleet entreprise !</h1>
</body>
</html>
```

Création du fichier de configuration de notre site Internet
```bash
nano /etc/nginx/sites-available/www7.starfleet.lan
```

Configuration du siteweb
```bash
server {

    listen 192.168.50.1:80;
    listen [::]:80;

    root /var/www/www7.starfleet.lan;

    index index.html;
    server_name www7.starfleet.lan;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

REndre actif notre site par la création de lien symbolique
````bash
ln -s /etc/nginx/sites-available/www7.starfleet.lan /etc/nginx/sites-enabled/www7.starfleet.lan
````

Vérifier si la syntaxe est bonne dans nginx
````
nginx -t
````

On redémarre les services (on peut aussi utiliser start et stop)
````
systemctl restart nginx
````

### 4.2 lancement de Nginx
```bash 
systemctl start nginx
```
## 5. Mise en place de PHP 7 et 8
### 5.1 Installation des sources
````
sudo apt-get update
````
Importation de la clé Sury 

````
curl -fsSL https://packages.sury.org/php/apt.gpg | gpg --dearmor -o /usr/share/keyrings/sury-php-archive-keyring.gpg
````

Ajout du dépôt 
````
echo "deb [signed-by=/usr/share/keyrings/sury-php-archive-keyring.gpg] https://packages.sury.org/php/ trixie main" > /etc/apt/sources.list.d/php.list
````


installation des versions 7.4 et 8.3

````
apt install php7.4-fpm php8.3-fpm
````

Modification de la configuration de notre site 
>nano /etc/nginx/sites-available/www7.starfleet.lan

Rajout du bloc : 
````
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
}

````
On redémarre et on relance 
````
nginx -t
systemctl restart nginx
````

Execution d'un script php  pour vérifier le bon fonctionnement de notre site 
>nano /var/www/www7.starfleet.lan/info.php
````
<?php
phpinfo(); 
?>
````


## 6.  Installation de PhpMYAdmin et MariaDb
### 6.1 Configuration 
#### 6.1.1 Installation PhpMyAdmin

````
cd /tmp
curl -LO https://www.phpmyadmin.net/downloads/phpMyAdmin-latest-all-languages.tar.gz
tar -xzf phpMyAdmin-latest-all-languages.tar.gz
mv phpMyAdmin-*-all-languages /var/www/php.starfleet.lan
````
>Choisir ni apache ni lighttpd 
> Choisir no pour la configuration de la base avec dbconfig-common

Création du dossier tmp dans le dossier /var/www/ pour le fonctionnement de phpmyadmin
````
mkdir -p /var/www/php.starfleet.lan/tmp
````
Donner les bonnes permissions à Nginx
````
chown -R www-data:www-data /var/www/php.starfleet.lan
chmod -R 755 /var/www/php.starfleet.lan
chmod 770 /var/www/php.starfleet.lan/tmp
````

Création du fichier de configuration
phpMyAdmin a besoin d'un fichier de configuration pour savoir :

- à quelle base de données se connecter (MariaDB en local, dans votre cas)
- comment sécuriser les sessions utilisateur (via la fameuse "clé secrète")

````
cp /var/www/php.starfleet.lan/config.sample.inc.php /var/www/php.starfleet.lan/config.inc.php
````
````
openssl rand -base64 32
````
Copier la clé générée dans  nano /var/www/php.starfleet.lan/config.inc.php
````
$cfg['blowfish_secret'] = 'mettre la clé là ';
````

Cette ligne dit à phpMyAdmin de se connecter à MariaDB sur la même machine (localhost)

````
$cfg['Servers'][$i]['host'] = 'localhost';
````


Mettre les extensions php nécessaire 

````
apt install php8.3-mysqli php8.3-mbstring php8.3-zip php8.3-gd php8.3-curl php8.3-xml -y
systemctl restart php8.3-fpm
````

#### 6.1.2 Installation MariaDB

Dépôt officiel de Maria DB 
````
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | bash
apt update
apt install mariadb-server mariadb-client -y
````

Checker la version
````
mariadb --version
apt policy mariadb-server
````

On paramètre pour lancer au démarrage mariadb
````
systemctl start mariadb
systemctl enable mariadb
````
### 6.2 Création du Vhost 

On créé un vhost pour php.starfleet.lan
````
nano /etc/nginx/sites-available/php.starfleet.lan
````
On paramètre : 
````
server {

    listen 443 ssl;

    listen [::]:443 ssl;

        root /var/www/php.starfleet.lan;
        index index.php index.html;
    server_name php.starfleet.lan;

    ssl_certificate /etc/nginx/ssl/starfleet.lan.crt;

    ssl_certificate_key /etc/nginx/ssl/starfleet.lan.key;


location  / {

try_files $uri $uri/ /index.php$is_args$args;
                }

location ~ \.php$ {

           include snippets/fastcgi-php.conf;

           fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;

           fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

           include fastcgi_params;
                }
}
````

On créé le lien symbolique
````
ln -s /etc/nginx/sites-available/php.starfleet.lan /etc/nginx/sites-enabled/php.starfleet.lan
````

Configuration de la base de donnée avec Maria
Lancer MySQL
````
mysql
````
creation du compte administrateur
````
CREATE USER 'phpadmin'@'localhost' IDENTIFIED BY 'adminmariadb';
````
````
GRANT ALL PRIVILEGES ON *.* TO 'phpadmin'@'localhost' WITH GRANT OPTION;
````
Savoir si un utilisateur existe déjà 
>SELECT User, Host FROM mysql.user WHERE User='phpadmin';

Sortir de Maria
````
EXIT;
````

Pour tester la connexion en CLI, On demande à se connecter à la base maria db avec l'utilisateur phpadmin et -p pour le mot de passe interactif 
````
mariadb -u phpadmin -p
````

### 7. Installation de Cockpit
 
Installer cockpit et activer le service 
````
apt install cockpit -y
systemctl enable --now cockpit.socket
````

Création du vhost pour cockpit
>nano /etc/nginx/sites-available/admin.starfleet.lan
````
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name admin.starfleet.lan;

    ssl_certificate     /etc/nginx/ssl/starfleet.lan.crt;
    ssl_certificate_key /etc/nginx/ssl/starfleet.lan.key;

    location / {
        proxy_pass https://127.0.0.1:9090/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_ssl_verify off;
    }
}
````

Création lien symbolique
````
ln -s /etc/nginx/sites-available/admin.starfleet.lan /etc/nginx/sites-enabled/admin.starfleet.lan
````

Vérifier et recharger 
````
nginx -t
systemctl reload nginx
````

### 8 LDAP 
#### 8.1 LDAP et accès aux pages web
````
apt install slapd ldap-utils -y
````
````
dpkg-reconfigure slapd
````

Répondre dans l'ordre 
````
DNS domain name → starfleet.lan
Organization name → Starfleet
Administrator password → (mot de passe fort ex:adminldap)
Confirm password → (le même)
Remove database when slapd is purged? → No
Move old database? → Yes
````

Pour faire une recherche dans ldap 
````
ldapsearch -x -H ldap://localhost -b "dc=starfleet,dc=lan"
````

Création de l'unité origranisationnelle persons
> nano /root/persons.ldif
````
dn: ou=persons,dc=starfleet,dc=lan
objectClass: organizationalUnit
ou: persons
````
On applique avec ldapadd
````
ldapadd -x -D "cn=admin,dc=starfleet,dc=lan" -W -f /root/persons.ldif
````

On rajoute l'utilsateur Spock

On va d'abord générer un mot de passe hachuré avec la commande : 
````
slappasswd
````
Puis on ajoute dans ce fichier : 
>nano /root/spock.ldif
````
objectClass: organizationalUnit
ou: persons
objectClass: shadowAccount
dn: uid=Spock,ou=persons,dc=starfleet,dc=lan
objectClass: inetOrgPerson
# Attribut schéma inetOrgPerson
cn: Spock
sn: Galac
givenName: Spock
uid: Spock
displayName : Spock Galac
mail: spock.galac@starfleet.lan
# Attribut schéma nis
objectClass: posixAccount
uidNumber: 20100
gidNumber: 10020
homeDirectory: /home/Spock
loginShell: /bin/bash
userPassword: {SSHA}y1RjbLGw44qgB+iE8N6Ba+lg8iGZKnHW
````

>Pour vérifier tous les caractère de nos fichier ldif !
````
cat -A /root/persons.ldif
````

#### 8.2 LDAP et configuration de la pagew web

Bien mettre son mot de passe en clair ici 
>nano /etc/nginx/sites-available/www8.starfleet.lan
````
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    root /var/www/www8.starfleet.lan;
    index index.html;

    server_name www8.starfleet.lan;

    ssl_certificate /etc/nginx/ssl/starfleet.lan.crt;
    ssl_certificate_key /etc/nginx/ssl/starfleet.lan.key;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    }

    location / {
        auth_request /auth-proxy;
        error_page 401 =200 /;
        try_files $uri $uri/ =404;
    }

    location = /auth-proxy {
        internal;
        proxy_pass http://127.0.0.1:8888;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_cache auth_cache;
        proxy_cache_valid 200 10m;
        proxy_cache_key "$http_authorization$cookie_nginxauth";

        proxy_set_header X-Ldap-URL         "ldap://ldap.starfleet.lan";
        proxy_set_header X-Ldap-Starttls    "false";
        proxy_set_header X-Ldap-BaseDN      "ou=persons,dc=starfleet,dc=lan";
        proxy_set_header X-Ldap-BindDN      "cn=admin,dc=starfleet,dc=lan";
        proxy_set_header X-Ldap-BindPass    "VOTRE_MOT_DE_PASSE_ADMIN_EN CLAIR !!!! ";
        proxy_set_header X-CookieName       "nginxauth";
        proxy_set_header Cookie             nginxauth=$cookie_nginxauth;
        proxy_set_header X-Ldap-Template    "(uid=%(username)s)";
        # ... la suite de tes headers LDAP
    }
}

````

#### 8.3 LDAP et deamon python

````
apt install python3 python3-pip git -y
cd /opt
git clone https://github.com/nginxinc/nginx-ldap-auth.git
cd nginx-ldap-auth
````

````
cd /opt/nginx-ldap-auth
pip3 install -r requirements.txt
````

````
python3 -c "import ldap; print('OK')"
````

````
apt install libldap2-dev libsasl2-dev python3-dev build-essential -y
pip3 install python-ldap --break-system-packages
````

Lancer le script 

````
python3 /opt/nginx-ldap-auth/nginx-ldap-auth-daemon.py
````

Relancer nginx

````
sudo nginx -t
````

Si le test n'est pas concluant car le dossier cache n'est pas connu alors faire : 
Ajouter dans le bloc http{...}
>nano /etc/nginx/nginx.conf

````
proxy_cache_path /var/cache/nginx/auth levels=1:2 keys_zone=auth_cache:10m2
````

Puis créer le dossier d2cache si besoin 

````
mkdir -p /var/cache/nginx/auth
chmod +x www-data:www-data /var/cache/nginx/auth
````

recharger ngninx
````
systemctl reload nginx
````

Modification des fichiers nginx pour chaque pages web (les blocs location/ et location = / auth-prox modifiés).
ici pour www7

````
server {

    listen 192.168.50.1:80;
    listen [::]:80;

    root /var/www/www7.starfleet.lan;
    index index.html;
    server_name www7.starfleet.lan;

location / {
    auth_request /auth-proxy;
    error_page 401 =200 /;
    try_files $uri $uri/ =404;
}

location = /auth-proxy {
    internal;
    proxy_pass http://127.0.0.1:8888;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_cache auth_cache;
    proxy_cache_valid 200 10m;
    proxy_cache_key "$http_authorization$cookie_nginxauth";

    proxy_set_header X-Ldap-URL      "ldap://ldap.starfleet.lan";
    proxy_set_header X-Ldap-Starttls "false";
    proxy_set_header X-Ldap-BaseDN   "ou=persons,dc=starfleet,dc=lan";
    proxy_set_header X-Ldap-BindDN   "cn=admin,dc=starfleet,dc=lan";
    proxy_set_header X-Ldap-BindPass "VOTRE_MOT_DE_PASSE_ADMIN";
    proxy_set_header X-CookieName    "nginxauth";
    proxy_set_header Cookie          nginxauth=$cookie_nginxauth;
    proxy_set_header X-Ldap-Template "(uid=%(username)s)";
}
}
````


### ProFTPd 
##### Notre configuration 

````
/etc/proftpd/
├── proftpd.conf                    ← config de base (identité serveur, port, IP d'écoute)
└── conf.d/
    ├── starfleet-ftp.conf          ← chroot, restrictions d'accès
    └── starfleet-tls.conf          ← chiffrement SSL/TLS
````

````
apt update
apt install -y proftpd
````

#### Configuration de base 

>nano /etc/proftpd/proftpd.conf 

````
ServerName "Starfleet FTP Server"
ServerType standalone
DefaultServer on
RequireValidShell off

DefaultAddress 192.168.50.1
UseIPv6 off
Port 21
Umask 022
MaxInstances 30

User proftpd
Group nogroup

Include /etc/proftpd/conf.d/
````


> nano /etc/proftpd/conf.d/starfleet-ftp.conf
````
DisplayLogin "Bienvenue sur le FTP Starfleet"

DefaultRoot /var/www ftpstarfleet

RootLogin off

MaxClients 5

<Limit LOGIN>
DenyGroup !ftpstarfleet
</Limit>
````



#### Ajout du certificat
````
nano /etc/proftpd/conf.d/starfleet-tls.conf
````

````
<IfModule mod_tls.c>
    TLSEngine                  on
    TLSLog                     /var/log/proftpd/tls.log
    TLSProtocol                TLSv1.2 TLSv1.3
    TLSOptions                 NoCert2quest
    TLSRSACertificateFile      /etc/n2nx/ssl/starfleet2.lan.cr    TLSRSACertificateKeyFile   /etc/nginx/ssl/st2fleet.lan.ke2    TLSVerifyClient            off
    TLSReq2red                on
</IfModule>
````
On réutilise le certificat pour nginx



##### Rendre le certificat lisible par Proftpd

````
ls -la /etc/nginx/ssl/starfleet.lan.key
````

````
chown root:nogroup /etc/nginx/ssl/starfleet.lan.key
chmod 640 /etc/nginx/ssl/starfleet.lan.key
````

#####  Vérification rapide de la cohérence de nos config
````
proftpd --configtest
````

### Ajout de l'utilisateur ftpuser  
````
addgroup ftpstarfleet
adduser ftpuser --shell /usr/sbin/nologin --home /var/www --ingroup ftpstarfleet --no-create-home
````
On donne un mot de passe à notre utilisateur
````
passwd ftpuser
````

````
chown -R www-data:ftpstarfleet /var/www
chmod -R 775 /var/www
````

##### On recharage le serveur ftp 

````
systemctl restart proftpd
systemctl status proftpd
````

#### Sur la VM cliente, on test avec le logiciel ftp 
````
apt install --download-only gftp -y
````

````
ls /var/cache/apt/archives/ | grep gftp
````

````
scp /var/cache/apt/archives/gftp*.deb utilisateur@IP_VM_CLIENTE:/tmp/
````

#### Correction du Path si besoin 
````
export PATH=$PATH:/usr/local/sbin:/usr/sbin:/sbin
````
#### Le rendre permanent 
````
echo 'export PATH=$PATH:/usr/local/sbin:/usr/sbin:/sbin' >> ~/.bashrc
````
#### Installation du paquet 
````
dpkg -i /var/cache/apt/archives/gftp*.deb
````
#### Utilsiation du logiciel dans le terminal 
````
gftp-text
````
````
set ftp_passive_transfer no
open ftp://192.168.50.1:21
````

#### Principales commandes 
````
ls              # lister le contenu distant
cd www8         # se déplacer dans un dossier
get fichier     # télécharger un fichier
put fichier     # envoyer un fichier
quit            # quitter
````


### 10 Installation de Visual Studio Code Server

````
wget -O- https://code-server.dev/install.sh | sh
````

##### Pour activer le service

````
systemctl enable --now code-server@root
systemctl status code-server@root
````

##### On configure les paramètre pour visual studio code server
>nano ~/.config/code-server/config.yaml
````
bind-addr: 127.0.0.1:8081
auth: password
password: 123456
cert: false
````

##### Ajouter le vhost ngninx
````
nano /etc/nginx/sites-available/vscore.starfleet.lan
````

````
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name vscore.starfleet.lan;

    ssl_certificate     /etc/nginx/ssl/starfleet.lan.crt;
    ssl_certificate_key /etc/nginx/ssl/starfleet.lan.key;

    location / {
       

        proxy_pass http://127.0.0.1:8081/;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Accept-Encoding gzip;
    }
}
````

##### On crée les liens symboliques 

````
ln -s /etc/nginx/sites-available/vscore.starfleet.lan /etc/nginx/sites-enabled/vscore.starfleet.lan
````

##### On relance nginx 
````
nginx -t
systemctl restart nginx
````


### 11 Pare-feu UFW 

Installation 
````
apt install ufw -y
````
On desactive tous les ports par défauts ENTRANTS
et on autorise tous ceux qui sont SORTANTS
````
ufw default deny incoming
ufw default allow outgoing
````
Puis on autorise les ports essentiels 
````
ufw allow 22/tcp      
ufw allow 53/tcp      
ufw allow 53/udp      
ufw allow 67/udp      
ufw allow 80/tcp      
ufw allow 443/tcp     
ufw allow 21/tcp      
ufw allow 389/tcp     
ufw allow 636/tcp     
````
Les ports concernant cockpit et visual studio code sont fermés car passant par nginx.


### 12 Genere le certificat avec mkcert

````
apt install libnss3-tools wget -y
wget -O /usr/local/bin/mkcert https://github.com/FiloSottile/mkcert/releases/latest/download/mkcert-v1.4.4-linux-amd64
chmod +x /usr/local/bin/mkcert
````
Création du certificat 
````
mkcert -install
````

Pour localiser le fichier 
````
mkcert -CAROOT
````

Création du dossier pour le certif dans nginx
````
mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl
````
````
mkcert -cert-file starfleet.lan.crt -key-file starfleet.lan.key starfleet.lan "*.starfleet.lan" www8.starfleet.lan www7.starfleet.lan php.starfleet.lan admin.starfleet.lan vscore.starfleet.lan
````

Aller dans le dossier où ils ont été créé 
````
ls -la /etc/nginx/ssl/
````
````
chmod 600 /etc/nginx/ssl/starfleet.lan.key
chmod 644 /etc/nginx/ssl/starfleet.lan.crt
````


````
curl -v https://www7.starfleet.lan
````

Envoi du certif à la VM cliente
````
scp "$(mkcert -CAROOT)/rootCA.pem" clientholodeck@192.168.50.70:/tmp/
````

Puis ajouter le certificat dans le navigateur internet de la vm Cliente (ici firefox) 
````
Paramètres -> Confidentialité et sécurité -> Certificats -> Afficher les certificats -> Autorités -> Importer
````
Fichier final 

>nano /etc/nginx/sites-available/www7.starfleet.lan

````
server {


    listen 443 ssl;
    listen [::]:443 ssl;
    root /var/www/www7.starfleet.lan; index index.html;

    server_name www7.starfleet.lan;

    ssl_certificate /etc/nginx/ssl/starfleet.lan.crt;
    ssl_certificate_key /etc/nginx/ssl/starfleet.lan.key;

location  / {

        try_files $uri $uri/ =404;
                }
}

````