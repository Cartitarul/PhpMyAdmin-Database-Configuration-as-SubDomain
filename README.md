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

#### With SSL
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
### Apache:
  1. Copy your apache file "FastDLL-Apache.conf" or "FastDLL-ApacheSSL.conf" in `etc/apache2/sites-available` or `/etc/httpd/conf.d/` (If on CentOS)
  2. Edit the DNS.DOMANIN.COM from the file to your Fully Qualified Domain Name (FQDN), you can find it in your node settings.

IF YOU TRY USING SSL
When using the SSL configuration you MUST create SSL certificates, otherwise your webserver will fail to start. Pterodacyl has a doc about this you can find it [HERE](https://pterodactyl.io/tutorials/creating_ssl_certificates.html#method-1:-certbot) .You need to certificate the **FQDN** in order to work !!!!

## Step 2 (SSH):
1. The following command assings the right permission for the module and to assign a user www.data for pterodactyl
  * ``` gpasswd -a www-data pterodactyl && chmod 755 /var/lib/pterodactyl/ && chmod 755 /var/lib/pterodactyl/volumes/ ```
2. Restart web configuration:
  ##### NGINX:
      systemctl restart nginx or nginx -s reload
  ##### Apache:
    * You do not need to run any of these commands on CentOS (For CentOS systemctl restart httpd ) *
    sudo ln -s /etc/apache2/sites-available/FastDLL-Apache.conf /etc/apache2/sites-enabled/FastDLL-Apache.conf  (For Apache NON SSL)
    sudo ln -s /etc/apache2/sites-available/FastDLL-ApacheSSL.conf /etc/apache2/sites-enabled/FastDLL-ApacheSSL.conf (For Apache WITH SSL)
    sudo a2enmod rewrite
    systemctl restart apache2
 ## Step 3 (Pannel edit):
 1. Edit the eggs install script with the following line at the very bottom end:
 *  ```  chmod 750 /mnt/server/ ```
 2. If you allready have installed a server and want to add FastDLL you need to run this command in SSH:
 * ``` chmod 755 /var/lib/pterodactyl/volumes/SERVER_UUID ``` Change the UUID to yours
 3. Configure our egg to use a JSON file parser:
 Edit the Configuration Files section from the egg config with the following: (For CSGO) Create a server.cfg file in csgo/cfg if you dont have one
 Please insert the node Link (yqdn) in *[NODE LINK]* and change http to https if you use SSL for the node subdomain.
 ```
 {
    "csgo/cfg/server.cfg": {
        "parser": "file",
        "find": {
            "sv_downloadurl": "sv_downloadurl \"http://[NODE LINK]/{{env.P_SERVER_UUID}}/csgo/\"",
            "sv_allowupload": "sv_allowupload \"0\"",
            "sv_allowdownload": "sv_allowdownload \"1\""
        }
    }
}
```
![alt text](https://i.imgur.com/4exzabq.png)
3. Name your node locations as our **FQDN** so it will act as a link or if you use only one node and onle location you can manualy edit the code and change {{env.P_SERVER_LOCATION}} in to your **FQDN** for example: 
  *   ```"sv_downloadurl": "sv_downloadurl \"http://DNS.DOMAIN.COM/{{env.P_SERVER_UUID}}/csgo/\"",```

4. For further error please contact me.

# Done
* The FastDownload url shoud look something like this http(s)://DNS.DOMAIN.COM/SERVER_UUID/csgo
* [Dr3Amer3r](https://github.com/Dr3Ame3r/pterodactyl_fastdl) - Credits Pterodactyl FastDL Version v1
