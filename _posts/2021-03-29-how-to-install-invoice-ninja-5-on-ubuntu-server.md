---
title: How to install Invoice Ninja on Ubuntu Server 20.04
author: darryl gibbs
date: 2021-03-29 -03:00
tags: [invoice ninja, selfhosted]
categories: [homelab, tutorial]
---
This is a tutorial I’ve written mostly for my own purposes that I have adapted from the orginal post from [TechnicallyComputers](https://forum.invoiceninja.com/t/install-invoice-ninja-v5-on-ubuntu-20-04/4588). Kudos to him!

## Prerequisites

* Fresh Ubuntu Server install on a VPS / VM with sudo access.

* Optional: A domain pointed at the server.

## Install necessary (and extra) software

First, update the system.

```sh
sudo apt upgrade -y && sudo apt dist-upgrade -y && sudo apt clean && sudo apt autoremove -y && sudo reboot
```

Then, install software we will use for the server. Note: Setup of fail2ban and UFW I’m going to leave to you. You can see my post on setting up Chamilo LMS where I cover that.

```sh
sudo apt install -y ufw \
fail2ban \
unzip \
zip \
git \
tree \
openssh-server \
net-tools \
curl \
software-properties-common
```

And some specific software we will need for this setup?

```sh
sudo apt install gcc g++ make php php-{fpm,bcmath,ctype,fileinfo,json,mbstring,pdo,tokenizer,xml,curl,zip,gmp,gd,mysqli} mariadb-server mariadb-client curl git nginx -y
```

Download and install PHP Composer

```sh
curl -sS https://getcomposer.org/installer -o composer-setup.php && \
sudo php composer-setup.php --install-dir=/usr/bin --filename=composer
```

## Setup Maria DB and the Database

Ok, now fire up MariaDB and also make it start up automatically with the system on reboot. THe last command will start the securing of the DB root account.

```sh
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation
```

Follow the below to know how to handle all the questions.

```sh
Enter current password for root (enter for none): strong-ass-MariaDB-root-password
Change the root password? [Y/n]: n
Remove anonymous users? [Y/n]: y
Disallow root login remotely? [Y/n]: y
Remove test database and access to it? [Y/n]: y
Reload privilege tables now? [Y/n]: y
```

```sh
sudo mysql -u root -p

CREATE DATABASE ninjadb;
CREATE USER 'ninja'@'localhost' IDENTIFIED BY 'ninjapassword';
GRANT ALL PRIVILEGES ON ninjadb.* TO 'ninja'@'localhost' IDENTIFIED BY 'ninjapassword' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

Make sure to update at least the password with your prefered choice, and the DB name and user should you wish to.


## Install Nginx (webserver)

First, let’s remove the stock Nginx page to avoid confusion.

```sh
sudo rm /etc/nginx/sites-enabled/default
```

Now let’s add our own .conf file specifically for our project

```sh
sudo nano /etc/nginx/conf.d/invoiceninja.conf
```

And now copy in the below, and edit as per your needs, namely: **port number** (normally this is set to port 80, but mine is set to something random for my own purposes), and **FQDN** (domain name).

```conf
server {
        listen 10000 default_server;
        listen [::]:10000 default_server;

        root /var/www/html/invoiceninja/public;

        # Add index.php to the list if you are using PHP
        index index.php index.html index.htm;
        client_max_body_size 50M;
        gzip on;
        gzip_types      application/javascript application/x-javascript text/javascript text/plain application/xml application/json;
        gzip_proxied    no-cache no-store private expired auth;
        gzip_min_length 1000;


        server_name invoice.example.com;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
        }


        if (!-e $request_filename) {
            rewrite ^(.+)$ /index.php?q= last;
        }

        # pass PHP scripts to FastCGI server
        #
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
        #
        #       # With php-fpm (or other unix sockets):
                fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        #       # With php-cgi (or other tcp sockets):
        #       fastcgi_pass 127.0.0.1:9000;
        }

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        location ~ /\.ht {
                deny all;
        }
}
```

The eagle eyed among you will see that it’s not complex, and that’s because I’ll have mine setup behind a reverse proxy. But if you need a complete .conf assumming that this is the sole purpose of the machine this is running on, see the config mentioned within the link at the beginning of the post.

Then start and enable Nginx after killing Apache.

```sh
sudo systemctl stop apache2
sudo systemctl disable apache2
sudo systemctl start nginx
sudo systemctl enable nginx
```

Test the nginx config to make sure all is well:

```sh
sudo nginx -t
```

## Get Invoice Ninja files

Visit the invoiceninja github release page and get the latest -release version zip file.

```sh
cd /var/www/html
sudo mkdir invoiceninja && cd invoiceninja
sudo wget https://github.com/invoiceninja/invoiceninja/releases/download/v5.1.32-release/invoiceninja.zip
sudo unzip invoiceninja.zip
```

### Installing and configuring software and dependencies

First move to the webroot for InvoiceNinja if not done so already.

```sh
cd /var/www/html/invoiceninja
```

Then run the compose. This is needed for InvoiceNinja to work.

```sh
sudo php /usr/bin/composer install --no-dev
```

If you’re short on memory, run this bad boy:    

```sh
sudo php -d memory_limit=-1 `which composer` install --no-dev
```

This will generate a .env file with an encryption key. Wait while this runs.

Then add that encryption key to the `.env`. Then run the auto config process. This must be repeated if files are moved or changed in the invoiceninja directory.

```sh
sudo php artisan key:generate
sudo php artisan optimize
```

Lastly, set folder permissions.

```sh
sudo chown -R www-data:www-data /var/www/html/invoiceninja
sudo chmod -R g+s /var/www/html/invoiceninja
```

InvoiceNinja requires some regular maintenance to take place. Failing to do so will place a bright red exclamation logo in the bottom left of the InvoiceNinja interface. This all takes place in `crontab -e`. Note: until you `reboot` the system, the exclamation will remain.

```sh
sudo -u www-data crontab -e
```

Insert this:

```sh
* * * * * php /var/www/html/invoiceninja/artisan schedule:run >> /dev/null 2>&1
```

Some folks have issues generating PDF’s. Should you have these issues, the below dependencies could fix that.

```sh
sudo apt-get install -y gconf-service libasound2 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgcc1 libgconf-2-4 libgdk-pixbuf2.0-0 libglib2.0-0 libgtk-3-0 libnspr4 libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 ca-certificates fonts-liberation libappindicator1 libnss3 lsb-release xdg-utils wget
```

Reboot.

```sh
sudo reboot
```

And finally, visit your IP/domain and complete the setup.

## Conclusion

That’s it! All should be working now.
