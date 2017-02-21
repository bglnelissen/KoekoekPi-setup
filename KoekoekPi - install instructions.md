## KoekoekPi install instructions for Raspbian Jessie

This script will get the KoekoekPi up and running from scratch

#### Steps to take

**Done**
- Put Jessie on the sd-card and enable `ssh`
- User setup
    - create 'normal' user
    - create 'server' user
    - change 'sudo' password
    - delete 'pi' user
- Setup `hostname`
- Enable `ssh` login via keys
- Use `apt-get` to update system
- webserver (MySQL, php7-fpm, nginx with ssl)
- owncloud
- git server
- apt-get installs
- mount USB drives at boot
- transmission

**ToDo**
- openvpn
- sickrage/sickbeard
- Fix: `perl: warning: Setting locale failed.`

#### Put Jessie on the sd-card and enable ssh

Get your SD-card inserted in your Mac. Download Raspbian Jessie image and write it to disk. Then add the `ssh` file in the `/boot` directory to enable `ssh` at boot.

```
# download Raspbian image
curl -O -J -L https://downloads.raspberrypi.org/raspbian_lite_latest

# extract the image (ZIP)
unzip 2017-01-11-raspbian-jessie-lite.zip

# create img variablename (IMG)
IMG=2017-01-11-raspbian-jessie-lite.img

# write the image (IMG) to disk and enable ssh
sudo date && diskutil list && echo "Enter the disk NUMBER and press [ENTER]" && read -p "(everything will be lost): " DISK && diskutil unmountDisk /dev/disk${DISK} && pv "${IMG}" | sudo dd of=/dev/rdisk${DISK} bs=100M && sleep 5 && touch /Volumes/boot/ssh && diskutil unmountDisk /dev/disk${DISK} && echo "Succes, you can remove the disk." || echo "Fail. - Re-insert disk and retry."
```

#### Ssh login

Boot the pi and login

```
# from your mac:
ssh pi@10.0.0.100 # password: raspberry
```

#### User setup

First create an account for the daily user `bas`.

```
# create user `bas`
sudo adduser bas
# add this user to the sudo group
sudo usermod -aG sudo bas
# list my current groups
groups bas
```

Create an account for our server activities named: `server`.

```
# create user `server`
sudo adduser server
# add user `server` to group `server`
sudo adduser server server
# list current groups
groups server
```

Change sudo password.

```
sudo passwd root
```

Delete user `pi`, you need to login by another account obviously.

```
sudo deluser pi
```

#### Install softwere we need for the setup

```
sudo apt-get -y install vim
```

#### Hostname setup

```
# change raspberrypi into koekoekpi
sudo vim /etc/hosts
# change raspberrypi into koekoekpi
sudo vim /etc/hostname
# activate changes
sudo /etc/init.d/hostname.sh && hostname
```

#### Enable login via ssh keys

ssh keys let you login without a password. Copy our local public key to the remote machine.

```
# from your mac:
cat ~/.ssh/id_rsa.pub | (ssh bas@koekoekpi "cat >> ~/.ssh/authorized_keys")
```

#### Update system, install essentials and reboot

Install updates and upgrades, also add packes for software you use a lot.

```
sudo apt-get -y update && sudo apt-get -y dist-upgrade && sudo reboot
```

#### Install webserver (MySQL, nginx, php7-fpm)

**Add sources**. This is a bit more tricky. We need to add the 'testing branch' aka 'stretch' for apt-get

```
# add the following line to add the 'stretch' branchs 
#   deb http://mirrordirector.raspbian.org/raspbian/ stretch main contrib non-free rpi
sudo vim /etc/apt/sources.list
```

We do not want all installs from the stretch branch, so we need to pin all packages to the 'jessie' release.

```
# Add the following:
#   Package: *
#   Pin: release n=jessie
#   Pin-Priority: 600
sudo vim /etc/apt/preferences

# update
sudo apt-get update
```

**MySQL**

```
# When the MySQL install runs you need to enter a root password
sudo apt-get -y install mysql-server mysql-client

# Now secure your MySQL install (remove anonymous users etc...), defaults should be fine
mysql_secure_installation
```

**Php7.0-fpm.** 

Install `php7.0-fpm` and atributes from the `stretch` branch.

```
sudo apt-get install -y --target-release stretch php7.0 php7.0-curl php7.0-mysql php7.0-gd php7.0-fpm php7.0-cli php7.0-opcache php7.0-mbstring php7.0-xml php7.0-zip
```

Setup the PHP7.0-fpm pool to use the `server` user and group.

```
# change the user and group to
#   user = server
#   group = server
# set permissions for unix socket
#   listen.owner = server
#   listen.group = server
# set your enviroment variables, uncomment (remove ';' ) the following lines
#   env[HOSTNAME] = $HOSTNAME
#   env[PATH] = /usr/local/bin:/usr/bin:/bin
#   env[TMP] = /tmp
#   env[TMPDIR] = /tmp
#   env[TEMP] = /tmp
sudo vim /etc/php/7.0/fpm/pool.d/www.conf
```

Test your installation

```
command -v php && php -v
```

**Encryption**.
We try to get strong encryption on our connections. Also see: *https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html*
We need generate a stronger DHE parameter.

```
# Generate a stronger DHE parameter which we use for nginx
# ! This is going to take a long time (hours)
cd /etc/ssl/certs && sudo openssl dhparam -out dhparam.pem 4096
```

Certbot is an easy-to-use automatic client that fetches and deploys 'Let’s Encrypt' SSL/TLS certificates for your webserver. We need to install this from source.

```
# as 'normal user' download cert-bot in ~/bin
mkdir -p ~/bin && cd ~/bin && wget https://dl.eff.org/certbot-auto && chmod a+x certbot-auto
```

Create some certificates.
In the future you can renew certificates doing a `certbot-auto renew`

```
# run certbot in automated mode, this will install a bunch and start the setup, answers like:
~/bin/certbot-auto --standalone -d guu.st --email b.g.l.nelissen@gmail.com certonly
```

Todo: add line to cron for automated updates.

**Nginx**. We use the 'apt-get stretch branch' setup above to install the latest nginx.

```
sudo apt-get install -y --target-release stretch nginx
```

Now we need to create the root directories `www` and `port8080`.

```
# create web directory (asif we are 'server')
sudo -u server mkdir /home/server/www && ls -la  /home/server/www
sudo -u server mkdir /home/server/port8080 && ls -la  /home/server/port8080
```

```
# create test index files
# echo "test <b>www</b><p><?php phpinfo(); ?>" | sudo -u server tee -a /home/server/www/index.html
# echo "test <b>port8080</b><p><?php phpinfo(); ?>" | sudo -u server tee -a /home/server/port8080/index.html
```

Replace the contents of /etc/nginx/nginx.conf 

```
#  # http://nginx.org/en/docs/ngx_core_module.html
#  user server server;
#  worker_processes 4;
#  pid /var/run/nginx.pid;
#  
#  events {
#    worker_connections  1024;
#  }
#  
#  http {
#    sendfile on;
#    tcp_nopush on;
#    tcp_nodelay on;
#    keepalive_timeout 65;
#    types_hash_max_size 2048;
#    client_max_body_size 1024m;
#    include /etc/nginx/mime.types;
#    default_type application/octet-stream;
#    gzip on;
#    gzip_disable "msie6";
#    include /etc/nginx/sites-enabled/*;
#  }
sudo vim /etc/nginx/nginx.conf
```

The server configuration file

```
#  # guu.st
#  server {
#    server_name guu.st;
#    listen 443 ssl;
#    root /home/server/www;
#    index index.php index.html;
#    # lets encrypt created keys
#    # renew keys using: sudo ./certbot-auto renew
#    # Lets encrypt needs .well-known
#    location ~ /.well-known {
#        allow all;
#    }
#    ssl on;
#    ssl_certificate /etc/letsencrypt/live/guu.st/fullchain.pem;
#    ssl_certificate_key /etc/letsencrypt/live/guu.st/privkey.pem;
#    # enforce strong encryption
#    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
#    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
#    ssl_prefer_server_ciphers on;
#    ssl_session_cache shared:SSL:10m;
#    ssl_dhparam /etc/ssl/certs/dhparam.pem;

#    # Add security related headers
#    add_header Strict-Transport-Security "max-age=15552000; includeSubDomains";
#
#    # Fast CGI
#    location ~ [^/]\.php(/|$) {
#      fastcgi_split_path_info ^(.+?\.php)(/.*)$;
#      if (!-f $document_root$fastcgi_script_name) {
#          return 404;
#      }
#      fastcgi_pass unix:/run/php/php7.0-fpm.sock;
#      fastcgi_index index.php;
#      include fastcgi_params;
#      fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
#    }
#    # deny acces to files/folders such as .htaccess, .htpasswd, .DS_Store, or starting with dot
#    location ~ ^/(data|config|\.ht|\.DS|db_structure\.xml|README|\.) {
#      deny all;
#      log_not_found off;
#      access_log off;
#    }
#  }
sudo vim /etc/nginx/sites-available/guu.st.conf
```

Add a server that is listening on port 8080

```
#  # port 8080
#  server {
#    server_name port8080;
#    listen 8080;
#    location / {
#      root /home/server/port8080/;
#    }
#    index.html;
#  }
sudo vim /etc/nginx/sites-available/port8080.conf
```

Redirect all port 80 trafic to port 443

```
#  # redirect port 80 to secure 443 ssl
#  server {
#    listen 80;
#    server_name guu.st;  
#    return 301 https://$server_name$request_uri;
#  }
sudo vim /etc/nginx/sites-available/redirect80to443.conf
```

Link all the configurations files from the 'available' directory to the 'enabled' directory

```
cd /etc/nginx/sites-enabled/
sudo ln -s ../sites-available/guu.st.conf ./
sudo ln -s ../sites-available/port8080.conf ./
sudo ln -s ../sites-available/redirect80to443.conf ./
```

Test configuration files

```
sudo nginx -t
```

**Start them up!**

```
# start nginx
sudo /etc/init.d/nginx restart
# start php7.0-fpm
sudo /etc/init.d/php7.0-fpm restart
```

**Check SSL security** https://www.ssllabs.com/ssltest/index.html


#### apt-get installs

list:

- vim, editor
- git-core, git server
- git, git client
- htop, process monitor.  #http://www.tecmint.com/install-htop-linux-process-monitoring-for-rhel-centos-fedora/
- lynx, CLI webbrowser

```
sudo apt-get -y update && sudo apt-get -y install \
  vim \
  git-core \
  git \
  htop \
  iotop \
  lynx

```

#### Mount USB drive

We need to setup the drive in the fstab file.

```
# Get drive info (UUID and /dev/location)
lsusb
sudo blkid
ls -l /dev/disk/by-uuid/

# Check the disk 
sudo fsck -f /dev/sda1

# create a mount location
sudo mkdir -p /media/KoekoekPi && sudo chown server:server /media/KoekoekPi && chmod 777 /media/KoekoekPi

# mount the drive manualy
sudo mount -o "umask=0,uid=nobody,gid=nobody" /dev/sda1 /media/KoekoekPi

# list mounted drives
df -Th
```

Add a line for the mount in **/etc/fstab**

```
#  # <device>                                <dir>             <type>  <options>     <dump> <fsck>
#  UUID=2446eac8-2de0-4872-8e4b-f3d782fe1550 /media/KoekoekPi  ext4    defaults,user 0      2
sudo vim /etc/fstab
```

Test the current **/etc/fstab** configuration, and when correct, run use it.

```
# Mount all
sudo mount --all --verbose
```

#### ownCloud

Download ownCloud and move it to `www`

```
# login as 'server'
sudo su server

# Check https://owncloud.org/install/#edition for the latest and greatest
cd ~/ && wget https://download.owncloud.org/community/owncloud-9.1.3.tar.bz2

# extract
tar jxfv owncloud-9.1.3.tar.bz2

# move 'owncloud' to www
mv owncloud ~/www/

# create a 'data' directory on the usb drive
sudo -u server mkdir /media/KoekoekPi/owncloud
```

Prepare MySQL with a user and a database

```
# login mysql as root and create new database + user
mysql -uroot -p
#   CREATE DATABASE IF NOT EXISTS owncloud;
#   GRANT ALL PRIVILEGES ON owncloud.* TO 'koekoek'@'localhost' IDENTIFIED BY 'Pa$$W0rd';
#   quit
```

Setup ownCloud web-configuration and ownCloud should be up and running.

```
# ownCloud setup via the browser on your Mac
# username [koekoek]
# password [create admin password]
# data [/media/KoekoekPi/owncloud]
```

When your owncloud data is not empty, you want to rescan all your files.

```
cd /home/server/www/owncloud
./occ files:scan --all
./occ files:clean
```

#### phpMyAdmin

Download phpMyAdmin and add it to 'www'

```
# login as 'server'
sudo su server

# Check https://www.phpmyadmin.net/downloads/ for the latest and greatest
cd ~/ && wget https://files.phpmyadmin.net/phpMyAdmin/4.6.5.2/phpMyAdmin-4.6.5.2-english.tar.bz2

# extract and delete tar
tar jxfv phpMyAdmin-4.6.5.2-english.tar.bz2 && rm phpMyAdmin-4.6.5.2-english.tar.bz2

# move 'phpMyAdmin' to 'www'
mv phpMyAdmin-4.6.5.2-english ~/www/phpmyadmin

# create a cookie blowfish secret
cd ~/www/phpmyadmin/ && cp config.sample.inc.php config.inc.php

# edit the blowfish_secret in config.inc.php
#   $cfg['blowfish_secret'] = 'superrandomgibrish@$#^%SDFGASDF@#$';
vim config.inc.php
```

Setup phpMyAdmin web-configuration (https://guu.st/phpmyadmin), login with root.

Create a new database user named server

1. Go to home
2. User accounts
3. 'Add user account'
4. User name 'server', Host name 'localhost', Password

#### OpenVPN server using piVPN

Install dependencies (which the script does not check...)

```
sudo apt-get -y install openvpn easy-rsa git iptables-persistent dnsutils expect unattended-upgrades
```

Run the PiVPN script

```
# run install script as normal user
curl -L https://install.pivpn.io | bash

# Now run 'pivpn add' to create the ovpn profiles.
# Run 'pivpn help' to see what else you can do!
# The install log is in /etc/pivpn.
# Vague01 Rheineck91 Rheineck89
```


#### Git server

Install git

```
sudo apt-get install -y wget git-core
```

Setup projects on the server, for example 'git.bin'

```
mkdir -p ~/git/bin.git && cd ~/git/bin.git
git init --bare
```

Setup your *local* machine

```
cd ~/bin
git init
git remote add KoekoekPi bas@guu.st:/home/bas/git/bin.git
```

Commit your code from local to pi using `git`, for example

```
git add . && git status
git commit -m "initial commit"
git push KoekoekPi master
```

#### Transmission

Install

```
sudo apt-get -y install transmission transmission-cli transmission-daemon
```

Setup

```
# create tmp/download dir
sudo -u server mkdir -p /media/KoekoekPi/Torrent/Complete
sudo -u server mkdir -p /media/KoekoekPi/Torrent/Incomplete

# stop transmission-daemon before changing settings
sudo service transmission-daemon stop

# change tmp/download dir in settings file
#   "blocklist-enabled": true, 
#   "blocklist-url": "http://list.iblocklist.com/?list=bt_level1",
#   "download-dir": "/media/KoekoekPi/Torrent/Complete",
#   "incomplete-dir": "/media/KoekoekPi/Torrent/Incomplete",
#   "rpc-authentication-required": false,
#   "rpc-whitelist": "127.0.0.1,10.0.*.*,192.168.*.*",
sudo vim /var/lib/transmission-daemon/info/settings.json

# change the user:group that runs the transmission-deamon
#   USER=server:server
# this might not work as expected... TODO
sudo vim /etc/init.d/transmission-daemon

# start transmission-daemon
sudo service transmission-daemon restart
```

Don't forget to open the port (e.g. '51413' TCP)

#### Citadel: Mail, Address book & Calendar server

Get to the latest and greatest before continuing and dependencies needed by Citadel.

```
sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y install gettext
```

Citadel is pre-configured so that IPv4 and IPv6 are set as the default transfer protocols.

```
# activate ipv6
sudo modprobe ipv6
```

Install Citadel

```
# get a cup of coffee...
# after a succesful install a setup will run, choose your setup.
curl http://easyinstall.citadel.org/install | sh
```

    - Citadel will be installed to `/usr/local/citadel`
    - WebCit will be installed to `/usr/local/webcit`
    - supporting libraries will be installed to `/usr/local/ctdlsupport`
    - The `setup` program will initialize (/usr/local/citadel/setup)
    - http://www.citadel.org/doku.php/installation:getting_started

Change the default ports away from 80 and 443, and reboot

```
# export WEBCIT_HTTPS_PORT='2000'
# export WEBCIT_HTTP_PORT='2001'
sudo vim /etc/default/webcit

# reboot to change settings and start te service
sudo reboot
```

Test the connection using `nc` as root

```
service nginx status
service webcit status

# as root
( printf "ECHO test\nQUIT\n"; sleep 3 ) | nc 127.0.0.1 504
```

The rest of the setup goes through the web interface.
Point your web browser at https://10.0.0.100:2000


- Administration
  - Edit site-wide configuration
    - General:
      - Fully qualified domain name: `guu.st`
      - Geographic location of this system: `Netherlands`
    - Settings:
      - Server IP address: `*`
      - XMPP: `-1`
    - POP3
      - POP3 listener port: `-1`
      - POP3 over SLL port: `-1`
  - Domain names and Internet mail configuration
    - Local host aliases: `guu.st` add
  - Add, change, delete user accounts
    - Add user
  - Restart now

When you login as a user you can setup forward rules

- Advanced
  - View/edit server-side mail filters
    - `if All forward to: example@mail.com`

---

**Extra's**

#### Resource usage

```
sudo iotop --only
```

#### Disk failures

Diskfailures do happen. For a clean guide check https://www.cyberciti.biz/tips/surviving-a-linux-filesystem-failures.html

```
# force checking of a drive
sudo e2fsck
```

#### Create backup image of the current system

Insert the SD in the Mac and run dd with gzip to create a backup image.

```
sudo date && diskutil list && read -p "Enter the disk NUMBER you want to backup and press [ENTER]: " DISK && diskutil unmountDisk /dev/disk${DISK} && sudo pv /dev/rdisk${DISK} | pigz -9 > koekoekpi."$(date +%Y%m%d)".img.gz
```

