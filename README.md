# Database PhpMyAdmin Installation Guide
Here you can find a simple PhpMyAdmin Database setup for Ubuntu using Apache.
This supports any of local or remote application (remote if the network and firewall configuration is corect)

# Recommended Dependencies 
## Optimal for Pterodactyl panel or any other:
##### Add "add-apt-repository" command 
    apt -y install software-properties-common curl apt-transport-https ca-certificates gnupg
##### Add additional repositories for PHP, Redis, and MariaDB
    LC_ALL=C.UTF-8 add-apt-repository -y ppa:ondrej/php
    add-apt-repository ppa:redislabs/redis -y
##### MariaDB repo setup script can be skipped on Ubuntu 22.04
    curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash
##### Update repositories list
    apt update
##### Add universe repository if you are on Ubuntu 18.04
    apt-add-repository universe
##### Install Dependencies
    apt -y install apache2 php8.1 php8.1-{common,cli,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip} mariadb-server tar unzip git redis-server php-json libapache2-mod-php php-mysql 

Keep in mind that you may need additional dependencies for additional content or even programs.

## Network configuration (UFW or any)
For the configuration to work the panel and DB needs these ports opened for communication:
    sudo ufw allow in "Apache"
    sudo ufw allow "Mysql"
    sudo ufw allow in 

## Installation
#### Installation:
    mkdir /var/www/phpmyadmin && cd /var/www/phpmyadmin
    wget https://www.phpmyadmin.net/downloads/phpMyAdmin-latest-english.tar.gz
    tar xvzf phpMyAdmin-latest-english.tar.gz
    mv /var/www/phpmyadmin/phpMyAdmin-latest-english/* /var/www/phpmyadmin
    
#### Post Install
    chown -R www-data:www-data * 
    mkdir config
    chmod o+rw config
    cp config.sample.inc.php config/config.inc.php
    chmod o+w config/config.inc.php

## Web Server Configuration
This guide only provides a configuration example for SSL secured installation. You can find documentation on how to obtain a certificate in the Official Documentation (https://pterodactyl.io/tutorials/creating_ssl_certificates.html)


### NGINX:
Create a file phpmyadmin.conf in /etc/nginx/sites-available/
Replace <domain> with your desired phpMyAdmin domain

#### NGINX With SSL
    server {
        listen 80;
        server_name <domain>;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name <domain>;

        root /var/www/phpmyadmin;
        index index.php;

        # allow larger file uploads and longer script runtimes
        client_max_body_size 100m;
        client_body_timeout 120s;

        sendfile off;

        # SSL Configuration
        ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
        ssl_session_cache shared:SSL:10m;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
        ssl_prefer_server_ciphers on;

        # See https://hstspreload.org/ before uncommenting the line below.
        # add_header Strict-Transport-Security "max-age=15768000; preload;";
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Robots-Tag none;
        add_header Content-Security-Policy "frame-ancestors 'self'";
        add_header X-Frame-Options DENY;
        add_header Referrer-Policy same-origin;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass unix:/run/php/php7.4-fpm.sock;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param HTTP_PROXY "";
            fastcgi_intercept_errors off;
            fastcgi_buffer_size 16k;
            fastcgi_buffers 4 16k;
            fastcgi_connect_timeout 300;
            fastcgi_send_timeout 300;
            fastcgi_read_timeout 300;
            include /etc/nginx/fastcgi_params;
        }

        location ~ /\.ht {
            deny all;
        }
    }
#### NGINX Without SSL
    server {
        listen 80;
        server_name <domain>;

        root /var/www/phpmyadmin;
        index index.php;

        # allow larger file uploads and longer script runtimes
        client_max_body_size 100m;
        client_body_timeout 120s;

        sendfile off;

        # See https://hstspreload.org/ before uncommenting the line below.
        # add_header Strict-Transport-Security "max-age=15768000; preload;";
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Robots-Tag none;
        add_header Content-Security-Policy "frame-ancestors 'self'";
        add_header X-Frame-Options DENY;
        add_header Referrer-Policy same-origin;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass unix:/run/php/php7.4-fpm.sock;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param HTTP_PROXY "";
            fastcgi_intercept_errors off;
            fastcgi_buffer_size 16k;
            fastcgi_buffers 4 16k;
            fastcgi_connect_timeout 300;
            fastcgi_send_timeout 300;
            fastcgi_read_timeout 300;
            include /etc/nginx/fastcgi_params;
        }

        location ~ /\.ht {
            deny all;
        }
    }
#### Applying nginx Configuration
    # You do not need to execute this file if you are using CentOS.
    sudo ln -s /etc/nginx/sites-available/phpmyadmin.conf /etc/nginx/sites-enabled/phpmyadmin.conf
Restart :
    systemctl restart nginx
### Apache:
Create a file phpmyadmin.conf in /etc/apache2/sites-available
Replace <domain> with your desired phpMyAdmin domain
For SSL keep in mind to have installed SSL for Apache2 for the virtual host
    sudo a2enmod ssl
#### Apache With SSL
    <VirtualHost *:80>
        ServerName <domain>
        RewriteEngine On
        RewriteCond %{HTTPS} !=on
        RewriteRule ^/?(.*) https://%{SERVER_NAME}/$1 [R,L]
    </VirtualHost>
    <VirtualHost *:443>
        ServerName <domain>
        DocumentRoot "/var/www/phpmyadmin"
        AllowEncodedSlashes On
        php_value upload_max_filesize 100M
        php_value post_max_size 100M
    <Directory "/var/www/phpmyadmin">
        AllowOverride all
    </Directory>
        SSLEngine on
        SSLCertificateFile /etc/letsencrypt/live/<domain>/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/<domain>/privkey.pem
    </VirtualHost>

#### Applying Apache Configuration
    sudo ln -s /etc/apache2/sites-available/phpmyadmin.conf /etc/apache2/sites-enabled/phpmyadmin.conf
    sudo a2enmod rewrite
    systemctl restart apache2

# Configuring
## Configuring DB 
Use this command and install a password and prompt everything else with 'n'
    sudo mysql_secure_installation
Check the php version to be 8.1
    php -v
Go and follow the instructions from 
    http(s)://domain/setup/index.php

# Finalizing
To finish after every step completed from above you need to delete the config folder, this is done to prevent anyone from being able to administrate phpMyAdmin in the future.
    rm -rf /var/www/phpmyadmin/config


# Conclusion  
Please keep in mind that this is not an official guide and is not supported by Pterodactyl as well. The guide is also proven to work in a non damaging manner. Proceed at own risk, I do not take responsibility for data loss! 
For any questions or issues please contact me and I will gladly help you. ;)