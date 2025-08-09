# Setup standard utilities

```bash

sudo apt-get update
sudo apt-get install git curl ca-certificates
```

# Setup default environment for PHP + PostgreSQL + NodeJS

### 1. Install `nginx`
```bash

sudo apt-get install nginx
```
This also creates a new user and group called `www-data`. 
Check if nginx works by opening the page in browser `http://192.168.2.149/`

### 2. Install `psql`
```bash

# Install psql
sudo apt-get install postgresql postgresql-client
# Connect to the database
sudo -u postgres psql
```
Create psql user and let him create new databases
```postgresql
CREATE USER myuser WITH PASSWORD 'mypassword';
ALTER USER myuser CREATEDB;
```
Create a new database (if needed)
```postgresql
CREATE DATABASE mydb OWNER myuser;
```


### 3. Install PHP and required libraries

```bash

sudo apt-get install php8.2-fpm \
    php8.2-pgsql \
    php8.2-amqp \
    php8.2-bcmath \
    php8.2-bz2 \
    php8.2-curl \
    php8.2-gd \
    php8.2-imagick \
    php8.2-imap \
    php8.2-intl \
    php8.2-ldap \
    php8.2-mbstring \
    php8.2-mcrypt \
    php8.2-memcached \
    php8.2-opcache \
    php8.2-redis \
    php8.2-soap \
    php8.2-xmlrpc \
    php8.2-xsl \
    php8.2-zip
```
Install composer
```bash

cd ~/
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
sudo chmod +x /usr/local/bin/composer
rm ~/composer-setup.php
```
### 4. Install Node.JS (LTS)
```bash

apt-get install nodejs npm
```

# Set up the project
### 1. Setup correct access permissions
```bash

# Go to the standard www directory
cd /var/www/
# remove the standard nginx html folder
sudo rm -rf ./html

# Create a new group to access /var/www
sudo groupadd wwwgroup
# Add the user to the group
sudo usermod -aG wwwgroup tatiana
# Change the group of /var/www
sudo chgrp -R wwwgroup /var/www
# Change access rules
sudo chmod -R 775 /var/www
# Make all files inside /var/www inherit the group
sudo chmod g+s /var/www
# Make sure all new files/dirs created by the user are also accessible to the groups the user belongs to
echo "umask 0002" >> ~/.bashrc

# Exit the machine and login back again
```
### 2. Pull the project from Git
```bash

cd /var/www/
git clone https://github.com/EventHorizon8/slots-bot.git ./
# Prepare the env
cp .env.dist .env
# Edit env parameters
nano .env

# Install dependencies
composer install


```
### 3. Setup Nginx to operate the site (_without SSL/HTTPS_)
```bash

sudo nano /etc/nginx/sites-available/default
```
Replace the contents with this:
```
server {

    # listen 443 ssl;
    listen 80 default_server;

    server_name	localhost;
    root	/var/www/web;

    index index.php;

    location ~* \.(woff|html|js|css|zip|jpeg|jpg|png|tgz|gz|rar|bz2|doc|xls|exe|pdf|ppt|tar|wav|bmp|rtf|swf|ico|flv|docx|xlsx|svg)$ {
        access_log off;
        try_files $uri $uri/ /index.php?$query_string;
    }

    location / {
        #try_files $uri $uri/ /index.php?$query_string;
        try_files $uri $uri/ /index.php$is_args$args;

    }

    location ~ \.php$ {
        fastcgi_connect_timeout 1;
        fastcgi_next_upstream timeout;
        # fastcgi_pass   localhost:9000;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        fastcgi_param  QUERY_STRING		$query_string;
        fastcgi_param  PATH_INFO		$fastcgi_script_name;
        fastcgi_param  REQUEST_METHOD	$request_method;
        fastcgi_param  CONTENT_TYPE		$content_type;
        fastcgi_param  CONTENT_LENGTH	$content_length;
        include        fastcgi_params;
    }

}
```
Restart Nginx
```bash

sudo service nginx restart
```

### 4. Setup CRON jobs under www-data (optional)
```bash

sudo crontab -u www-data -e
# edit the file according to your needs
# ...

# You can check the status of the cron jobs by running
sudo journalctl | grep CRON
```
Example cron entry (pay attention that it changes the directory first):
```
0 23 * * * cd /var/www && /usr/bin/php yii check-changes
```
