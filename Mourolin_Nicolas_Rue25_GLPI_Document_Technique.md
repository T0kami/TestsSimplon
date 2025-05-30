# Brief partie 4 - Installer GLPI 10 sur une machine virtuelle Debian 11.6

Contexte : L’agence **Rue 25** souhaite déployer GLPI comme solution open-source de gestion de parc. Cette documentation décrit chaque étape afin que tout administrateur système puisse reproduire l’installation.

----------

## Prérequis

 - **Une VM Debian 11.6** déjà installée

Nous partons du principe que la machine est déjà configurée et installée dans VirtualBox.
Configuration minimale : 1 vCPU, 2 Go de RAM et 20 Go de stockage

- **Un compte avec `sudo`**

Pour exécuter les commandes d’installation sans passer en root.

- **Config Réseau**

Configurer le port forwarding dans VirtualBox, pour accéder au SSH et à l'interface web de GLPI depuis votre machine hôte.
Dans la configuration VirtualBox onglet Réseau : mode **NAT** + règles de port forwarding 
- Hôte : 
2222 => VM : 22 pour SSH
8080 => VM : 80 pour GLPI

Tutoriel si besoin : https://www.activecountermeasures.com/port-forwarding-with-virtualbox/

- **(Facultatif) Nom de domaine local**

Exemple : `glpi.rue25.local`, pratique pour se connecter via un nom plutôt qu’une adresse IP.

----------

## 1 – Installation du serveur SSH

Pour installer le client & serveur SSH 

```bash
sudo apt install openssh-client openssh-server
```
Le service se lance automatiquement. Pour vérifier son statut, vous pouvez exécuter cette commande :
```bash
sudo systemctl status ssh
```

Testez la connexion depuis votre machine hôte :
```bash
ssh -p 2222 utilisateur@127.0.0.1   # 2222 si vous avez fait le port-forwarding
```
> Si la connexion fonctionne vous pouvez exécuter toutes les étapes suivantes à distance, ce qui est plus confortable que via VirtualBox.

----------
## 2 – Installer la stack Web (LAMP)

GLPI a besoin d’un serveur Web (Apache), d’une base de données (MariaDB) et de PHP. Le trio s’appelle **LAMP** (Linux Apache MySQL/MariaDB PHP).

Pour installer LAMP et les extensions PHP requises :
```bash
sudo apt install apache2 mariadb-server \
  php libapache2-mod-php \
  php-{mysqli,mbstring,curl,gd,simplexml,intl,ldap,apcu,xmlrpc,cas,zip,bz2,imap}
```
 **Redémarrer Apache**  
 Les nouvelles extensions PHP seront prises en compte après un redémarrage :  
`sudo systemctl restart apache2`

----------

## 3 – Configurer la base de données

MariaDB propose un petit assistant pour installer rapidement une base de données
```bash
sudo mysql_secure_installation
```
-   Un mot de passe **root** vous sera demandé : choisissez quelque chose de solide.
-   Vous pouvez répondre par Yes  (Y) aux diverses questions pour une configuration standard (supprimer utilisateurs anonymes, désactiver root à distance, etc)
    
Accédez ensuite au shell mysql
```bash
sudo mysql -u root -p
```

Et collez les commandes SQL suivantes :
```sql
CREATE DATABASE db_glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'username'@'localhost' IDENTIFIED BY 'MotDePasseFort!';
GRANT ALL PRIVILEGES ON db_glpi.* TO 'username'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
- Remplacez "username" par le nom utilisateur de votre choix, évitez les noms génériques par mesure de sécurité.
- Générez un mot de passe sécurisé et unique.

----------

## 4 – Télécharger et copier GLPI

Téléchargez l’archive
    
```bash
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/10.0.18/glpi-10.0.18.tgz
```
    
Décompressez 
    
```bash
tar -xzvf glpi-10.0.18.tgz
```
    
Créez le dossier de destination
    
```bash
sudo mkdir -p /var/www/glpi   
```
    
Copiez les fichiers
    
```bash
sudo cp -R ./glpi/. /var/www/glpi/
sudo chown -R www-data /var/www/glpi   
```
Le compte **www-data** est celui qu’utilise Apache. En lui donnant les droits d'accès au dossier, GLPI pourra s'exécuter normalement.

----------

## 5 – Configurer Apache pour GLPI

Apache doit savoir que quand quelqu’un visite votre serveur il faut l’amener vers GLPI. Cela se fait via un "VirtualHost".

Créez le fichier config

```bash
sudo nano /etc/apache2/sites-available/glpi.conf
```
    
Collez le contenu suivant (adaptez le nom de domaine si besoin) :
    
```html
    <VirtualHost *:80>
        ServerName glpi.rue25.local      # Votre domaine local
    
        DocumentRoot /var/www/glpi/public
    
        <Directory /var/www/glpi/public>
            Require all granted
		    RewriteEngine  On
		    
		    # Passe les headers d'authorisation à PHP
		    RewriteCond  %{HTTP:Authorization}  ^(.+)$
			RewriteRule  .*  -  [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
            
            # Redirige presque tout vers index.php, indispensable à GLPI
            RewriteCond %{REQUEST_FILENAME} !-f
            RewriteRule ^(.*)$ index.php [QSA,L]
        </Directory>
    </VirtualHost>
```
    
Activer les modules requis et relancer Apache
    
```bash
sudo a2enmod rewrite        # Active le module rewrite si ce n'est pas déjà fait
sudo a2ensite glpi.conf     # Active le nouveau fichier de configuration
sudo a2dissite 000-default.conf		# Supprime la config par défaut

sudo systemctl restart apache2
```
À ce stade, Apache sert déjà les fichiers GLPI, mais l’application n’est pas encore « installée » (il manque la configuration web).

----------

## 6 – Finir l’installation dans le navigateur

1.  Ouvrez votre navigateur depuis la machine hôte à l'adresse suivante :
    
    -   soit `http://127.0.0.1:8080 (ou autre selon le port choisi dans VirtualBox)`
    -   soit `http://glpi.rue25.local/` selon le domaine configuré dans Apache
        
2.  L’assistant GLPI apparaît. Suivez les indications:
    
    -   **Langue** => Français
    -   **Licence** => Accepter.
    -   **Installer** (et non "Mettre à jour")
    -   **Vérification** des pré-requis : tout doit s’afficher en vert.
    -   **Base de données** :
        -   hôte = `localhost`
        -   utilisateur = `nom choisi`
        -   mot de passe = celui que vous avez choisi
        -   base = `db_glpi`.
            
3.  Une fois terminé, GLPI vous donne des comptes par défaut (glpi, tech...) 
**> **Pensez à changer ces mots de passe immédiatement** via Administration => Utilisateurs**
----------

## 7 – Nettoyer et sécuriser après l’installation

Supprimez le fichier d’installation
```bash
sudo rm -f /var/www/glpi/install/install.php
```
----------

## Pour aller plus loin

-   **HTTPS** : un certificat gratuit Let’s Encrypt se configure facilement avec certbot.
    
-   **Sauvegardes** :
    
    -   la base MariaDB
    -   le dossier GLPI (`/var/www/glpi`)  
        Programmez un cron quotidien et stockez la copie en dehors de la VM.
    
-   **Plugins** : GLPI possède de nombreux modules à installer au besoin.
-  **Configurez un pare-feu**
- Et plus encore...

----------

## Félicitations !

Notre Administrateur en herbe peut à présent disposer de son installation complète de GLPI. Et je peux finir cet exercice pour prendre une pause bien méritée!
