NeXTCloud Service
=================

The nextcloud container configuration  is based on the [wonderfall/nextcloud](https://hub.docker.com/r/wonderfall/nextcloud/): I remind you some provided build features:

+ Based on Alpine Linux Edge.
+ Bundled with nginx and PHP 7.1.
+ Automatic installation using environment variables.
+ Package integrity (SHA512) and authenticity (PGP) checked during building process.
+ Data and apps persistence.
+ OPCache (opcocde), APCu (local) installed and configured.
+ system cron task running.
+ MySQL, PostgreSQL (server not built-in) and sqlite3 support.
+ Redis, FTP, SMB, LDAP, IMAP support.
+ GNU Libiconv for php iconv extension (avoiding errors with some apps).
+ No root processes. Never.
+ Environment variables provided (see below).


Environment variables
=========

+ UID : nextcloud user id
+ GID : nextcloud group id
+ UPLOAD_MAX_SIZE : maximum upload size (default : 10G)
+ APC_SHM_SIZE : apc memory size (default : 128M)
+ OPCACHE_MEM_SIZE : opcache memory size in megabytes (default : 128)
+ CRON_PERIOD : time interval between two cron tasks (default : 15m)
+ CRON_MEMORY_LIMIT : memory limit for PHP when executing cronjobs (default : 1024m)
+ TZ : the system/log timezone (default : Etc/UTC)
+ ADMIN_USER : username of the admin account (default : none, web configuration)
+ ADMIN_PASSWORD : password of the admin account (default : none, web configuration)
+ DOMAIN : domain to use during the setup (default : localhost)
+ DB_TYPE : database type (sqlite3, mysql or pgsql) (default : sqlite3)
+ DB_NAME : name of database (default : none)
+ DB_USER : username for database (default : none)
+ DB_PASSWORD : password for database user (default : none)
+ DB_HOST : database host (default : none)

Port 8888 : HTTP Nextcloud port of the nginx webserver built inside the container.

Volumes
=======

    /data : Nextcloud data.
    /config : config.php location.
    /apps2 : Nextcloud downloaded apps.
    /nextcloud/themes : Nextcloud themes location.

Database
========

This ansible role is built on the mariadb container; you can try to run it manually using the
following:

docker pull mariadb:latest

    docker run -d --name db_nextcloud \
       -v /mnt/nextcloud/db:/var/lib/mysql \
       -e MYSQL_ROOT_PASSWORD=supersecretpassword \
       -e MYSQL_DATABASE=nextcloud -e MYSQL_USER=nextcloud \
       -e MYSQL_PASSWORD=supersecretpassword \
       mariadb:10


Pull the image and create a container. /mnt can be anywhere on your host, this is just an example. Change MYSQL_ROOT_PASSWORD and MYSQL_PASSWORD values (mariadb). You may also want to change UID and GID for Nextcloud, as well as other variables (see Environment Variables).


    docker run -d --name nextcloud \
        --link db_nextcloud:db_nextcloud \
        -v /mnt/nextcloud/data:/data \
        -v /mnt/nextcloud/config:/config \
        -v /mnt/nextcloud/apps:/apps2 \
        -v /mnt/nextcloud/themes:/nextcloud/themes \
        -e UID=1000 -e GID=1000 \
        -e UPLOAD_MAX_SIZE=10G \
        -e APC_SHM_SIZE=128M \
        -e OPCACHE_MEM_SIZE=128 \
        -e CRON_PERIOD=15m \
        -e TZ=Etc/UTC \
        -e ADMIN_USER=mrrobot \
        -e ADMIN_PASSWORD=supercomplicatedpassword \
        -e DOMAIN=cloud.example.com \
        -e DB_TYPE=mysql \
        -e DB_NAME=nextcloud \
        -e DB_USER=nextcloud \
        -e DB_PASSWORD=supersecretpassword \
        -e DB_HOST=db_nextcloud \
        wonderfall/nextcloud:10.0


Reverse proxy
========

Now you have to use a reverse proxy in order to access to your container through Internet.
Of course you can use your own solution to do so! nginx, Haproxy, Caddy, h2o, there's plenty 
of choices and documentation about it on the Web.

If you're using nginx, there are two possibilites :

+ __nginx is on the host__ : get the Nextcloud container IP address with docker inspect nextcloud | grep IPAddress\" | head -n1 | grep -Eo "[0-9.]+". But whenever the container is restarted or recreated, its IP address can change. Or you can bind Nextcloud HTTP port (8888) to the host (so the reverse proxy can access with http://localhost:8888 or whatever port you set), but in this case you should consider using a firewall since it's also listening to http://0.0.0.0:8888.

+ __nginx is in a container__ : you can link nextcloud container to an nginx container so you can use 
proxy_pass http://<yourdomain>:8888. 

Here there is an example of configuration:

    server {
        listen 8000;
        server_name example.com;
        return 301 https://$host$request_uri;
    }

    server {
        listen 4430 ssl http2;
        server_name example.com;

        ssl_certificate /certs/example.com.crt;
        ssl_certificate_key /certs/example.com.key;

        include /etc/nginx/conf/ssl_params.conf;

        client_max_body_size 10G; # change this value it according to $UPLOAD_MAX_SIZE

        location / {
            proxy_pass http://<yourdomain>:8888;
            include /etc/nginx/conf/proxy_params;
        }
    }


Run the ansible role to setup the containers
==========

Run the provided role to configure the nextcloud container:

    ansible-playbook site.yml -i hosts --limit nextcloud

N.B. The group_vars/all contains all env variables described above.

Let's Encrypt
=========

To make nginx work in HTTPS mode, we need to provide the {key:cert} values to the configuration
file; to do this there are two ways:

+ provide a self signed certificate using openssl:

        openssl req -x509 -newkey rsa:2048 -keyout key.pem -out crt.pem

+ Request a certificate and make it to be signed by a valid CA: for this reason I use certbot to create a letsencrypt cert and I finally add a task in the playbook that configure the cron job for the cert renew launching @daily:

        certbot renew --quiet

