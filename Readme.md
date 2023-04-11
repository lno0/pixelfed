# Read me 
This is a fork from [Pixelfed](https://github.com/pixelfed/pixelfed).
This fork is organized in a slightly different way in order to make the installation of Pixelfed with Docker a little easier. 

## Pre-requisites:
---
You will need a Linux server with ```git```, ```docker``` and ```nginx```. 

While ```git``` is included wiht most Linux distributions you will have to install both ```docker``` and ```nginx``` on your own. 

See the instructions:
* [Docker - How to install](https://docs.docker.com/engine/install/)
* Nginx - Can almost always be installed from your package manager. 

You will aslo need the packages ```php8.1-cli``` and ```php-fpm```, install these from the package manager from your distro. 

For the SSL certificates we will be using Certbot, see the official website for installation instructions.

* [Certbot](https://certbot.eff.org/)

## Installation:
------------

First things first. 

Log into your server and go the directory where you want to place Pixelfed. Once there clone the repository to your server: 
```bash
git clone https://github.com/lno0/pixelfed
```

Once this is done, place yourself inside the repository:
```bash
cd pixelfed/
```

Now you should start by configuring your .env file. You can do this with the command
```bash
nano .env
```

Inside the .env file you can define and set several parameters for your installation. 

Make sure to get the following right: 
```bash
APP_NAME="Pixelfed Prod"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://real.domain
APP_DOMAIN="real.domain"
ADMIN_DOMAIN="real.domain"
SESSION_DOMAIN="real.domain"
```
Replace the value of ```APP_NAME``` with the name of your instance. 
Replace ```APP_URL```,```APP_DOMAIN``` and ```SESSION_DOMAIN``` with your Fully Qualified Domain Name. 

Next, on the section below:
```bash
## Databases (MySQL)
DB_CONNECTION=mysql
DB_DATABASE=pixelfed_prod
DB_HOST=db
DB_PASSWORD=pixelfed_db_pass
DB_PORT=3306
DB_USERNAME=pixelfed
# Pass the same values as above
MYSQL_DATABASE=pixelfed_prod
MYSQL_PASSWORD=pixelfed_db_pass
MYSQL_RANDOM_ROOT_PASSWORD=true
MYSQL_USER=pixelfed
```

Fill in the information of your database. ```DB_DATABASE``` is the name of your database, ```DB_USERNAME``` and ```DB_PASSWORD``` are the login parameters for the database admin user. 

On the next section: 
```bash
## Cache (Redis)
REDIS_CLIENT=phpredis
REDIS_SCHEME=tcp
REDIS_HOST=redis
REDIS_PASSWORD=redis_password
REDIS_PORT=6379
REDIS_DATABASE=0
```

Make sure to define a ```REDIS_PASSWORD```.

For information on the other parameters of the .env file and the possible configurations for each one of them please check [Pixelfed - Generic Installation Guide](https://docs.pixelfed.org/running-pixelfed/installation/). 

Once this is done you can build your image with the command: 
```bash
docker build . -t pixelfed:latest 
```

Once the image is built all you have to do is bring it up: 
```bash
docker compose up -d
```

If everything has gone as planned you will now have four running containers. 

## Post installation steps:
---
After the containers are setup you will need to do a few things in order to get Pixelfed running on your server. 

First run the command: 
```bash
docker compose exec app php artisan key:generate
```

This will write the application key in your .env file. You can check this with ```cat .env``` you will see a base64 encoded key in front of ```APP_KEY=```

Next you need to migrate the databases, do this with 
```bash
docker compose exec app php artisan migrate --force
```
You can now import the cinties into your database, in that way you can selec them as locations on your posts.
```bash
docker compose exec app php artisan import:cities
```

If you enabled ActivityPub you will need to run this command:
```bash
docker compose exec app php artisan instance:actor
```

If you enabled OAuth (necessary if you wnat to use a mobile app, for instance) you need to run this:
```bash
docker compose exec app php artisan passport:keys
```

Then cache your routes and views:
```bash
docker compose exec app php artisan route:cache
docker compose exec app php artisan view:cache
```

And finally cache your configuration (you need to do this everytime you change your .env file):
```bash
docker compose exec app php artisan config:cache
```

The only thing left to do on the server side is setup Nginx, get your certificates and create a user. 

## Setting up Nginx:
---

To get Nginx going you will need to setup a virtual host, you can do this with an .conf file, see below an example of a Nginx.conf file: 

```
map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

server {
    listen 80;
    listen [::]:80;
    server_name YOUR_DOMAIN_NAME;
    root PATH_TO_PIXELFED/PUBLIC;
    # enforce https
    location /.wellknown/acme-challenge/ { allow all; }
    location / {return 301 https://$host$request_uri;}
}


server {

    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name YOUR_DOMAIN;

    ssl_protocols TLSv1.2;
    ssl_ciphers HIGH:!MEDIUM:!LOW:!aNULL:!NULL:!SHA;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;

    ssl_certificate /etc/letsencrypt/live/YOUR_DOMAIN_NAME/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/YOUR_DOMAIN_NAME/privkey.pem; # managed by Certbot

    keepalive_timeout 70;
    sendfile on;
    charset utf-8;
    client_max_body_size 100M;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";
    add_header Strict-Transport-Security "max-age=31536000";

    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:PATH_TO_PHPFPM;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name; # or $request_filename
    }

    location / {
        #try_files $uri/ /index.php?$query_string;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $http_x_forwarded_host;
        proxy_set_header X-Forwarded-Port $http_x_forwarded_port;
        proxy_redirect off;
        proxy_pass   http://127.0.0.1:8080;
    }
}




```

Create this file with a name like ```your-domain-name.conf``` (where ```your-domain-name``` is your fully qualified domain name) and place it in ```/etc/nginx/sites-available/```

Replace ```YOUR_DOMAIN_NAME``` with your domain name. 

Replace ```PATH_TO_PIXELFED/PUBLIC``` with the absolute path to your public folder, usually this should be something like ```/home/"your-username"/pixelfed/public```. 

Replace ```PATH_TO_PHPFPM``` with the path to your fpm.sock file. This is distro dependent but you can search for the file with a ```find / -iname "*.sock"```

When this is done create a softlink of this file into the nginx sites-enabled folder: 

```bash
ln -s /etc/nginx/sites-available/"your-domain".conf /etc/nginx/sites-enabled/"your-domain".conf
```
Replace here ```"your-domain"``` with your domain name.

## Getting the SSL certificates
---

We will be using Certbot for this. This is really simple, use the follwoing command:

```bash 
certbot certonly
```

It will give you two options, select option number one. 
Answer the questions asked. When asked for your domain name make sure that you will write it just like you did on the nginx file. Once the certificates are created it will show you the location of the files, this should be the one already in your nginx file, but if for some reason this is not so make sure to changet the path in your nginx file in the seciton ```ssl_certificate```

## Creating a user 
---
To create a user go back to your /pixelfed directory and use the command: 
```bash
docker compose exec app php artisan user:create
```

Fill in the required information. 

## Launching Nginx:
---
Now all you have to do is launch Nginx with: 
```bash
sudo systemctl start nginx
```


Surf after this to your website to make sure everything is working 
