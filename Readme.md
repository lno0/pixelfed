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
# pass the same values to the db itself
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

The only thing left to do on the server side is setup Nginx and create a user. 

