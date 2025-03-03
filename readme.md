# Containerized Wordpress

This Repo is meant to be used to quickly & painlessly spin up a new containerized Wordpress instance on a new vps. 

## Environment

Two files are needed, one for the db name `db.env`, and another `wp.env`

`db.env`
```
MYSQL_ROOT_PASSWORD=you_root_password
MYSQL_USER=your_wp_db_username
MYSQL_PASSWORD=your_wp_db_password
MYSQL_DATABASE=your_wp_db
```

`wp.env`
```
WORDPRESS_DB_HOST=database:3306
WORDPRESS_DB_NAME=wordpress
WORDPRESS_DB_PASSWORD=wp_db_user
WORDPRESS_DB_USER=wp_db_user
WORDPRESS_DEBUG=1
WORDPRESS_TABLE_PREFIX=wh_
```

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes. See deployment for notes on how to deploy the project on a live system.

  1. clone the repo
  2. cd compose-wordpress
  3. vim .env files and add your environment varibles
  4. sed s/&DOMAIN&/your_domain/g ./nginx-conf/nginx-nossl.conf > ./nginx/nginx.conf
  5. sed -i s/&DOMAIN&/your_domain/g ./docker-compose.yaml
  6. sed -i s/&EMAIL&/your_email/g ./docker-compose.yaml
  7. docker-compose up -d will execute the containers

You will see output confirming that your services have been created:

```
Output
Creating database ... done
Creating wordpress ... done
Creating webserver ... done
Creating certbot   ... done
```
Using docker-compose ps, check the status of your services:

```
docker-compose ps
```

If everything was successful, your database, wordpress, and webserver services will be Up and the certbot container will have exited with a 0 status message:
```
Output
  Name                 Command               State           Ports       
-------------------------------------------------------------------------
certbot     certbot certonly --webroot ...   Exit 0                      
database          docker-entrypoint.sh --def ...   Up       3306/tcp, 33060/tcp
webserver   nginx -g daemon off;             Up       0.0.0.0:80->80/tcp 
wordpress   docker-entrypoint.sh php-fpm     Up       9000/tcp    
```

## Setting up SSL

You can now check that your certificates have been mounted to the webserver container with docker-compose exec:

```
docker-compose exec webserver ls -la /etc/letsencrypt/live
```

If your certificate requests were successful, you will see output like this:

```
Output
total 16
drwx------    3 root     root          4096 May 10 15:45 .
drwxr-xr-x    9 root     root          4096 May 10 15:45 ..
-rw-r--r--    1 root     root           740 May 10 15:45 README
drwxr-xr-x    2 root     root          4096 May 10 15:45 your_domain
```

To enable `https` follow the steps:
  * replace the --staging flag in 'certbot' in your docker-compose file with --force-renewal
  * docker-compose up --force-recreate --no-deps certbot
  * docker stop webserver
  * curl -sSLo nginx/options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
  * rm nginx/nginx.conf
  * sed s/&DOMAIN&/your_domain/g ./nginx-conf/nginx-ssl.conf > ./nginx/nginx.conf
  * add "443:443" port mapping to the webserver service in the docker-compose file
  * docker-compose up -d --force-recreate --no-deps webserver
 
Check your services with docker-compose ps:

```
docker-compose ps
```

You should see output indicating that your database, wordpress, and webserver services are running:

```
Output
  Name                 Command               State                     Ports                  
----------------------------------------------------------------------------------------------
certbot     certbot certonly --webroot ...   Exit 0                                           
db          docker-entrypoint.sh --def ...   Up       3306/tcp, 33060/tcp                     
webserver   nginx -g daemon off;             Up       0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
wordpress   docker-entrypoint.sh php-fpm     Up       9000/tcp    
```
With your containers running, you can now complete your WordPress installation through the web interface.

## Renewing the certificates

To setup the automatic renewal of the certificates follow the steps:
 * chmod +x ssl_renew.sh
 * sudo crontab -e
 * add '`0 12 * * * ~/compose-wordpress/scripts/ssl_renew.sh >> /var/log/cron.log 2>&1`'

## Create Swap file

To create a swap file and mount it:
 * sudo fallocate -l 1G /swapfile
 * sudo dd if=/dev/zero of=/swapfile bs=1024 count=1048576
 * sudo chmod 600 /swapfile
 * sudo mkswap /swapfile
 * sudo swapon /swapfile
 * sudo vim /etc/fstab
 * add `/swapfile swap swap defaults 0 0`
 * sudo mount -a

## Configuration

Install additional php utilities `sudo apt install php-curl php-gd php-mbstring php-xml php-xmlrpc`

Edit php.ini to handle upload sizes and timeouts:
 * 