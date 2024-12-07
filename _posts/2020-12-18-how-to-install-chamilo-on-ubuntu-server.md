---
title: How to Install Chamilo on Ubuntu Server
author: darryl gibbs
date: 2020-12-18
tags: [chamilo, selfhosted]
categories: [homelab, tutorial]
---
This is a tutorial I’ve written mostly for my own purposes that I have adapted from the orginal post from [Vultr](https://www.vultr.com/docs/how-to-install-chamilo-1-11-8-on-ubuntu-18-04-lts). Kudos to those guys.

## Prerequisites

* Fresh Ubuntu Server install on a VPS with sudo access.
* A domain pointed at the server.

## Install necessary (and extra) software

First, update the system.
```sh
sudo apt upgrade -y && sudo apt dist-upgrade -y && sudo apt autoremove -y && sudo apt clean -y && sudo reboot
```

Then, install software we will use for the server.

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

## Update UFW firewall rules
```sh
sudo ufw allow in ssh && \
sudo ufw allow in http && \
sudo ufw allow in https && \
sudo ufw enable
```

## Install, setup and secure MariaDB

To install the latest version of MariaDB directly, you will need to add the MariaDB repos. Confirm the latest version [here](https://mariadb.org/download/), and update the version number and Ubuntu version as needed.
```sh
sudo apt install -y software-properties-common && \
sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8 && \
sudo add-apt-repository 'deb [arch=amd64,arm64,ppc64el] http://mirrors.accretive-networks.net/mariadb/repo/10.5/ubuntu focal main' && \
sudo apt update && \
sudo apt install -y mariadb-server
```

During the install, provide a strong password for MariaDB. If not requested, no sweat, we’ll secure it in the next block.

Ok, now fire up MariaDB and also make it start up automatically with the system on reboot:
```sh
sudo systemctl start mariadb.service
sudo systemctl enable mariadb.service
```
Now, let’s secure MariaDB:
```sh
sudo /usr/bin/mysql_secure_installation
```
A prompt will come up, answer as below, but remember to provide a **strong** root password now if not done so previously.
```sh
Enter current password for root (enter for none): strong-ass-MariaDB-root-password
Change the root password? [Y/n]: n
Remove anonymous users? [Y/n]: y
Disallow root login remotely? [Y/n]: y
Remove test database and access to it? [Y/n]: y
Reload privilege tables now? [Y/n]: y
```

## Install Apache
```sh
sudo apt install -y apache2
```
Go to the server URL to confirm Apache is up and running.

Remove the standard Apache welcome page.
```sh
sudo mv /var/www/html/index.html /var/www/html/index.html.old
```
Backup the default .conf file and block Apache from exposing the web root directory:
```sh
sudo cp /etc/apache2/apache2.conf /etc/apache2/apache2.conf.bak
sudo sed -i "s/Options Indexes FollowSymLinks/Options FollowSymLinks/" /etc/apache2/apache2.conf
```
Enable the Apache rewrite module, start the service and auto-start
```sh
sudo a2enmod rewrite
sudo systemctl start apache2.service
sudo systemctl enable apache2.service
```

## Install latest PHP Packages

In order to run the latest PHP 7.x packages, we need to add the repo:
```sh
sudo add-apt-repository -y ppa:ondrej/php
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```
Install the latest 7.4 packages. Before doing so, confirm the latest version [here](https://www.php.net/downloads.php), and if higher the 7.4, substitute that version wherever you see 7.4.
```sh
sudo apt install -y php7.4 php7.4-opcache php7.4-cli php7.4-curl php7.4-common php7.4-gd php7.4-intl php7.4-mbstring php7.4-mysql libapache2-mod-php7.4 php7.4-soap php7.4-xml php7.4-xmlrpc php7.4-zip php7.4-ldap php-apcu-bc
```
Backup and then edit the PHP config file pertaining to Apache:
```sh
sudo cp /etc/php/7.4/apache2/php.ini /etc/php/7.4/apache2/php.ini.bak
sudo sed -i 's#;date.timezone =#date.timezone = America/Sao_Paulo#' /etc/php/7.4/apache2/php.ini
```
Right, now let’s adjust the php.ini file to meet the necessities of Chamilo LMS:
```sh
sudo nano /etc/php/7.4/apache2/php.ini
```
Find these:
```sh
session.cookie_httponly =
upload_max_filesize = 2M
post_max_size = 8M
```
And update them to look like these:
```sh
session.cookie_httponly = 1
upload_max_filesize = 100M
post_max_size = 100M
```
## Install Chamilo
### Setup the Database
```sh
sudo mysql -u root -p

CREATE DATABASE chamilo;
CREATE USER 'chamilo'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON chamilo.* TO 'chamilo'@'localhost' IDENTIFIED BY 'password' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```
Make sure to update at least the **password** with your prefered choice, and the DB name and user should you wish to.
Get Chamilo files

Let’s now head to [Chamilo’s GitHub page](https://github.com/chamilo/chamilo-lms/releases) to confirm we have the latest version. Update the link and steps below as needed:
```sh
cd
wget https://github.com/chamilo/chamilo-lms/releases/download/v1.11.14/chamilo-1.11.14.tar.gz
```
Extract all the files to the /opt directory:
```sh
sudo tar -zxvf chamilo-1.11.14.tar.gz -C /opt
```
Create a symbolic link between the unzip directory and the webserver root directory"
```sh
sudo ln -s /opt/chamilo-lms-1.11.14 /var/www/html/chamilo
```
Modify the ownership of all Chamilo files to the `www-data` user and the `www-data` group:
```sh
sudo chown -R www-data:www-data /opt/chamilo-lms-1.11.14
```
Configure Apache virtual server for Chamilo
```sh
cat <<EOF | sudo tee /etc/apache2/sites-available/chamilo.conf
<VirtualHost *:80>
ServerAdmin admin@example.com
DocumentRoot /var/www/html/chamilo
ServerName chamilo.example.com
ServerAlias example.com
<Directory />
AllowOverride All
Require all granted
</Directory>
<Directory /var/www/html/chamilo>
Options FollowSymLinks
AllowOverride All
Require all granted
</Directory>
ErrorLog /var/log/apache2/chamilo.example.com-error_log
CustomLog /var/log/apache2/chamilo.example.com-access_log common
</VirtualHost>
EOF
```
Update the **ServerAdmin**, **ServerName**, **ServerAlias**, **ErrorLog** and **CustomLog** as necessary.

Then, create the symbolic links to the /etc/apache2/sites-enabled directory, while deleting the Apache default link.
```sh
sudo rm /etc/apache2/sites-enabled/000-default.conf
sudo ln -s /etc/apache2/sites-available/chamilo.conf /etc/apache2/sites-enabled/
```
Finally, restart Apache so all the changes are activated.
```sh
sudo systemctl restart apache2.service
```

## Complete the install in the browser

Open up your browser and navigate to your server, for example: http://chamilo.iamamazing.com, and click on Install Chamilo. From here on, fill out the details as requested, but here is a quick guide for you.

* `Step 1 - Installation Language`: Choose the language of choice and then click the `Next` button.
* `Step 2 – Requirements`: Make sure that all mandatory requirements have been met, and then click the `New installation` button.
* `Step 3 – Licence`: Read the licence if you dare, and the click the checkbox next to the `I agree` sentence, fill in all required fields, and then click the Next button.
* `Step 4 – MySQL database settings`: Input the database credentials we setup earlier and then click the `Check database connection` button to verify them. Click the Next button to move on.
* `Step 5 – Config settings`: Modify the pre-set administrator password, fill in the other fields accordingly, and then click the `Next` button.
* `Step 6 – Last check before install`: Review all of the settings and hit the `Install Chamilo` button to start the web installation.
* `Step 7 – Installation process execution`: When Chamilo is successfully installed, click the `Go to your newly created portal`. button to finish the web installation wizard.

### Post-Install Safety Measure: DO NOT SKIP THIS!!!

In order to tighten security of the installation folders, we need to change their permissions:
```sh
sudo chmod -R 0555 /var/www/html/chamilo/app/config
sudo rm -rf /var/www/html/chamilo/main/install
```

## Fail2Ban

Sourced from: [How to Secure Your Linux Server with fail2banHow to Secure Your Linux Server with fail2ban](https://www.howtogeek.com/675010/how-to-secure-your-linux-computer-with-fail2ban/)

Firstly, install fail2ban.
```sh
sudo apt-get install fail2ban
```
### Configuring fail2ban

fail2ban creates ‘jails’ in .conf files. This standard config is removed each time fail2ban upgrades, so we’ll make a customized file.

So, lets make a copy of the default file and then edit our own version.
```sh
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```
Scroll down to the **[DEFAULT]** and **[sshd]** sections, that come up from about line 90 onwards. It appears near the top too, but ignore that. Find the following lines and update as below. See the source link above for further details of each.

[DEFAULT] section:
```sh
ignoreip = 127.0.0.1/8 ::1

bantime = 24h

findtime = 5m

maxretry = 3
```
[sshd] section: **ADD** these 2 lines to the end of the [sshd] section.
```sh
maxretry = 3
enable = true
```

### Enable fail2ban

Enable the service and auto-start.

sudo systemctl enable fail2ban
sudo systemctl start fail2ban

Check that fail2ban is working.
```sh
sudo systemctl status fail2ban.service
```
Check fail2ban status. This should show the active jail.
```sh
sudo fail2ban-client status
```
To show the exact status of the jail we activated. It will show any IPs that are banned.
```sh
sudo fail2ban-client status sshd
```
Testing the jail: see source link above.


### Unbanning an IP

After seeing the banned IP from above, use the command below, inserting the appropriate IP.
```sh
sudo fail2ban-client set sshd unbanip 192.168.4.25
```
Now it can try again.


## Let’s Encrypt Certificate

This part is adapted from a [Digital Ocean tutorial](https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-20-04).
### Certbot setup

SSL certs are a **must have** so let’s hook them up.

Firstly, install the certbot.

sudo apt install -y certbot python3-certbot-apache

Confirm your `ServerName` and `ServerAlias` details in your `chamilo.conf` file as we will need these for the cert request.
```sh
sudo nano /etc/apache2/sites-available/chamilo.conf
```
Let’s quickly recheck that our Apahce files are all good and then reload Apache incase we made changes.
```sh
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Allow HTTPS through the firewall

Confirm the current status.
```sh
sudo ufw status
```
If HTTP and HTTPS aren’t allowed through, please refer back to the UFW setup above.
Obtain SSL Certificate
```sh
sudo certbot --apache
```
A script will start to configure SSL. You will be asked to give an email, and you must in order to receive notices of expiring certs.

When asked, agree to Let’s Encrypt’s TOS, by pressing A.

Certbot will then ask you which domains you want to get certs for. This is where having accurate ServerName and ServerAlias details is important.

When asked whether you want to redirect traffic, chose option 2 for YES. This will force all conenctions over HTTPS.
```sh
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 2
```
If you get the message below, you’re golden!
```sh
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations! You have successfully enabled https://your_domain and
https://www.your_domain

You should test your configuration at:
https://www.ssllabs.com/ssltest/analyze.html?d=your_domain
https://www.ssllabs.com/ssltest/analyze.html?d=www.your_domain
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/your_domain/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/your_domain/privkey.pem
   Your cert will expire on 2020-07-27. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot again
   with the "certonly" option. To non-interactively renew *all* of
   your certificates, run "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```
You can confirm your SSL cert and it’s grade over at [SSL Labs Server Test](https://www.ssllabs.com/ssltest/).


### Verify Certbot Auto-Renewal

Our certbot package will automatically auto-renew our certificate. To check the status of this service, let’s use:
```sh
sudo systemctl status certbot.timer
```
Your output should look like this.
```sh
Output
● certbot.timer - Run certbot twice daily
     Loaded: loaded (/lib/systemd/system/certbot.timer; enabled; vendor preset: enabled)
     Active: active (waiting) since Tue 2020-04-28 17:57:48 UTC; 17h ago
    Trigger: Wed 2020-04-29 23:50:31 UTC; 12h left
   Triggers: ● certbot.service

Apr 28 17:57:48 fine-turtle systemd[1]: Started Run certbot twice daily.
```
And if you want to test the renewal process, run this.
```sh
sudo certbot renew --dry-run
```

## Conclusion

That’s it! All should be working now.
