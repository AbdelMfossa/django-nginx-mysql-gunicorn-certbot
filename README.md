# Déploiement d’une application Django sur un serveur Ubuntu

## Cas d’utilisation
Supposons que nous ayons un projet Django (portefolio) en local, utilisant SQLite3 comme base de données, que nous avons poussé sur GitHub. Nous obtenons un serveur VPS avec l’IP : `51.52.53.54` et nous rattachons le domaine (enregistrement A) `abdelmfossa.com` à notre IP du VPS. Notre base de données sur MySQL s’appellera `base`. Nous supposons également que nous utiliserons un fichier `.env`.

## Structure du projet
```
portefolio/
├── manage.py
├── .env
├── .gitignore
├── db.sqlite3
├── requirements.txt
├── portefolio/
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── app1/
│   ├── migrations/
│   ├── templates/
│   ├── static/
│   ├── tests/
│   └── ...
├── app2/
│   └── ...
└── ...
```


## I. Mise en place de la base de données
La première étape cruciale dans le déploiement d’une application Django consiste à configurer la base de données. Dans cette section, nous allons explorer comment installer et configurer MySQL sur votre serveur, créer une base de données et définir les autorisations d’accès. La base de données joue un rôle central dans le stockage des données de votre application, et sa configuration correcte est essentielle pour garantir le bon fonctionnement de votre projet. Suivez les étapes attentivement pour mettre en place votre base de données de manière sécurisée et efficace. 🛢️

1. Connectez-vous au serveur MySQL :
```
sudo mysql
mysql -u root -p
```
2. Créez la base de données :
```
CREATE DATABASE base;
```
3. Créez un utilisateur et accordez-lui les privilèges :
```
CREATE USER 'abdel'@'localhost' IDENTIFIED WITH mysql_native_password BY 'mot_de_passe';
GRANT ALL PRIVILEGES ON base.* TO 'abdel'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```
4. Quittez MySQL :
```
exit
```
[Lien d'exemple](https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-20-04)


## II. Contenu du fichier .env au niveau du serveur
Le fichier `.env` est un élément clé dans la configuration de votre application Django lors du déploiement. Il contient des variables d’environnement spécifiques à votre serveur et à votre application. Voici le contenu de notre fichier .env :

```
SECRET_KEY=django-insecure-+mbguqio_t=yf1sgwi^^f4)sxan7s%bz#ih-v+44p6#dgsv_!*
DEBUG=False
ALLOWED_HOSTS=abdelmfossa.com, localhost, 51.52.53.54
DB_NAME=base
DB_USER=abdel
DB_PASSWORD=mot_de_passe
DB_HOST=localhost
DB_PORT=3306
```


## III. Déploiement
1. Connectez-vous au serveur :
```
ssh -i .ssh/id_rsa abdel@51.52.53.54 -p 2223
```
2. Clonez le projet :
```
sudo su
cd /var/www/
git clone git@djbcjecbjbcjec.git
cd portefolio
```
3. Créez un environnement virtuel et installez les dépendances :
```
python3 -m venv env
source env/bin/activate
pip install -r requirements.txt
pip install gunicorn
```
4. Modifiez les paramètres dans `nano portefolio/portefolio/settings.py` :
```
ALLOWED_HOSTS = ['abdelmfossa.com', '51.52.53.54', 'localhost']
STATIC_URL = 'static/'
import os
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```
5. Appliquez les migrations et collectez les fichiers statiques :
```
python manage.py makemigrations
python manage.py migrate
python manage.py collectstatic
```
6. Modifiez les permissions :
```
cd ..
chown www-data: -R portefolio/
```
7. Configurez Gunicorn :
    * Créez `nano /etc/systemd/system/gunicorn.socket` :
        ```
        [Unit]
        Description=gunicorn socket

        [Socket]
        ListenStream=/run/gunicorn.sock

        [Install]
        WantedBy=sockets.target
        ```
    * Créez `nano /etc/systemd/system/gunicorn.service` :
        ```
        [Unit]
        Description=gunicorn daemon
        Requires=gunicorn.socket
        After=network.target

        [Service]
        User=root
        Group=www-data
        WorkingDirectory=/var/www/portefolio
        ExecStart=/var/www/portefolio/env/bin/gunicorn \
                --access-logfile - \
                --workers 3 \
                --bind unix:/run/gunicorn.sock \
                portefolio.wsgi:application

        [Install]
        WantedBy=multi-user.target
        ```
    * Activez et démarrez Gunicorn :
        ```
        sudo systemctl start gunicorn.socket
        sudo systemctl enable gunicorn.socket
        sudo systemctl status gunicorn.socket
        file /run/gunicorn.sock
        sudo systemctl daemon-reload
        sudo systemctl restart gunicorn
        ```
8. Configurez Nginx :
    * Créez un fichier de configuration pour votre domaine (`nano /etc/nginx/sites-available/abdelmfossa.com`) :
        ```
        server {
            listen 80;
            server_name abdelmfossa.com www.abdelmfossa.com;

            location = /favicon.ico { access_log off; log_not_found off; }
            location /static/ {
                root /var/www/portefolio;
            }

            location / {
                include proxy_params;
                proxy_pass http://unix:/run/gunicorn.sock;
            }
        }
        ```
    * Activez la configuration :
        ```
        sudo ln -s /etc/nginx/sites-available/abdelmfossa.com /etc/nginx/sites-enabled
        sudo nginx -t
        sudo systemctl restart nginx
        ```
    * Autorisez le trafic Nginx dans le pare-feu :
        ```
        sudo ufw delete allow 8000
        sudo ufw allow 'Nginx Full'
        ```
9. Redémarrez Gunicorn et vérifiez l’état du service :
```
sudo systemctl restart gunicorn
sudo systemctl daemon-reload
sudo systemctl restart gunicorn.socket gunicorn.service
```
10. Vérifiez à nouveau la configuration Nginx et redémarrez-le :
```
sudo nginx -t && sudo systemctl restart nginx
```
Votre application Django devrait maintenant être déployée avec succès sur votre serveur Ubuntu ! Bonne continuation ! 🚀<br>
[Lien d'exemple](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu)


## VI. Certbot SSL
1. Installez Certbot et le plugin Nginx :
```
sudo apt install certbot python3-certbot-nginx
```
2. Autorisez le trafic Nginx dans le pare-feu :
```
sudo ufw allow 'Nginx Full'
```
3. Supprimez l’autorisation pour le trafic HTTP (port 80) :
```
sudo ufw delete allow 'Nginx HTTP'
```
4. Vérifiez l’état du pare-feu :
```
sudo ufw status
```
5. Obtenez un certificat SSL pour votre domaine (remplacez abdelmfossa.com par votre propre domaine) :
```
sudo certbot --nginx -d abdelmfossa.com -d www.abdelmfossa.com
```
6. Testez le renouvellement du certificat (mode à sec) :
```
sudo certbot renew --dry-run
```
Votre site devrait maintenant être sécurisé avec un certificat SSL grâce à Certbot. 🔒<br>
[Lien d'exemple](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04)


## V. Migrer les données de Sqlite3 vers MySql
1. Exportation des données depuis SQLite3.
Utilisez la commande suivante pour exporter vos données vers un fichier JSON (db.json) en excluantt les données relatives aux autorisations et aux types de contenu, car elles sont spécifiques à SQLite3 :
```
python manage.py dumpdata --exclude auth.permission --exclude contenttypes > db.json
```
2. Nettoyage de la base de données MySQL (facultatif).
Si vous souhaitez supprimer toutes les données de votre base de données MySQL, utilisez la commande :
```
python manage.py flush
```
3. Importation des données dans MySQL.
Assurez-vous que votre base de données MySQL est configurée correctement (comme expliqué précédemment).
Utilisez la commande suivante pour charger les données depuis le fichier JSON dans MySQL :
```
python manage.py loaddata db.json
```
Votre application Django devrait maintenant utiliser la base de données MySQL avec les données migrées depuis SQLite3. N’oubliez pas d’adapter ces étapes à votre propre projet. 🚀
