## KoekoekPi install instructions for Raspbian

This script will get the KoekoekPi up and running from scratch.

#### Steps

USB drive setup

- Put Raspbian from SD on USB
- Enable USB boot
- Enable SSH

Raspbian setup

- User setup
	- create 'normal' user
	- create 'server' user
	- change 'sudo' password
	- delete 'pi' user
- Setup `hostname`
- Enable `ssh` login via keys
- Use `apt-get` to update system
- Enable USB boot
- RAID setup
- webserver (MySQL, php7-fpm, nginx with ssl)
- owncloud
- git server
- apt-get installs
- mount USB drives at boot
- transmission
- git
- openvpn
- sickrage/sickbeard
- Citadel: Mail, Address book & Calendar server
- wondershaper
- homekit
- RAID storage

## USB drive setup

#### Put Raspbian on an SD (we will later move it to USB)

Get your SD-card inserted in your Mac. Download Raspbian Jessie image and write it to disk. Then add the `ssh` file in the `/boot` directory to enable `ssh` at boot.

```
# download Raspbian image
# curl -O -J -L  https://downloads.raspberrypi.org/raspbian_lite_latest
curl -O -J -L https://downloads.raspberrypi.org/raspbian_lite_latest

# extract the image (ZIP)


# write the image (IMG) to disk and enable ssh (error illigal value: use bs=100m or bs=100M)
FILE='2017-11-29-raspbian-stretch-lite' && unzip "${FILE}.zip" && sudo date && diskutil list && echo "Enter the disk NUMBER and press [ENTER]" && read -p "(everything will be lost): " DISK && diskutil unmountDisk /dev/disk${DISK} && pv "${FILE}".img | sudo dd of=/dev/rdisk${DISK} bs=100m && sleep 5 && touch /Volumes/boot/ssh && diskutil unmountDisk /dev/disk${DISK} && echo "Succes, you can remove the disk." || echo "Fail. - Re-insert disk and retry."
```

#### Enable SSH

SSH is disabled by default. To enable it create a file named `ssh` on `/boot`

```
# enable ssh at boot
touch /Volumes/boot/ssh
```

#### Enable USB boot

Before a Raspberry Pi 3 will boot from a mass storage device, it needs to be booted from an SD card with a config option to enable USB boot mode. This will set a bit in the OTP (One Time Programmable) memory in the Raspberry Pi SoC that will enable booting from a USB mass storage device. Once this bit has been set, the SD card is no longer required. Note that any change you make to the OTP is permanent and cannot be undone.

```
# Add this line in /boot/config.txt
program_usb_boot_mode=1
```

The boot sequence can be to fast for the USB drives to startup properly. A boot delay might solve this. On the Pi or on another device edit the `/boot/config.txt`.

```
# Add this line in /boot/config.txt
# boot_delay=10
```

We prefer ehternet only and like to disable WLAN.

```
# Add this line in /boot/config.txt
# dtoverlay=pi3-disable-wifi
```

Do not leave a trailing empty line in `/boot/config.txt`

---

## Setup Raspbian

Insert the drive in the Pi and boot.

#### SSH login

Boot the pi and login

```
# from your mac:
ssh pi@10.0.0.100 # password: raspberry
```

#### User setup

First create an account for the daily user `bas`.

```
# create user `bas` and enter password.
sudo adduser bas
# add this user to the sudo group
sudo usermod -aG sudo bas
# list the current groups for bas
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

Delete user `pi`, first you need to login by another account ... obviously.

```
sudo deluser pi
```


#### Major update routine

This wil break most custom setups. Do not run this like an animal.

```
sudo apt-get -y update && sudo apt-get -y upgrade && sudo apt-get -y autoremove && sudo reboot
```

#### Install software we need for the setup

```
sudo apt-get -y install pv vim nano git-core git && sudo apt-get -y autoremove
```

#### Hostname setup

```
# change raspberrypi into koekoekpi
sudo vim /etc/hosts
# change raspberrypi into koekoekpi
sudo vim /etc/hostname
```

#### Enable login via ssh keys

ssh keys let you login without a password. Copy our local public key to the remote machine.

```
# from your mac:
# create your ssh key if needed `ssh-keygen`
# ssh bas@koekoekpi
ssh bas@doma.in # ssh-keygen
cat ~/.ssh/id_rsa.pub | (ssh bas@doma.in "cat >> ~/.ssh/authorized_keys")
```

#### Create RAID partition

Install the RAID controller and setup mdadm configuration and error reporting.

```
# install
sudo apt-get -y update && sudo apt-get -y install mdadm && sudo apt-get -y autoremove
# setup
sudo dpkg-reconfigure mdadm
```

Create 2 identical partitions on the free space of the USB drives. We create a new partition with the unused space on the drive (50GB). Because of compatibility reasons we want to use all the free space except for the last 100MB.

*A sector is an X number of bytes. Check the number of bytes per sector, and check the start sector of the largest unused space.*

```
sudo fdisk /dev/sda
# press 'F' - list free unpartitioned space
# Sector size (logical/physical): 512 bytes / 512 bytes
#    Start       End   Sectors  Size
#     2048      8191      6144    3M
# 15523840 121307135 105783296 50.5G
```

We want 104857600 bytes (100MB) free at the end, so we need to calculate how many sections are in 100MB. `104857600/512=204800 sectors is 100MB`, so the last sector is `121307135-204800=121102335`.

To enable RAID on boot the partition type must be changed from 'Linux' to 'Linux raid autodetect', aka type '83' to type 'fd'.

```
# press 'F' - list free unpartitioned space
# press 'n' - add a new partition
# - 'p' (default)
# - '3' (default)
# - First sector: 15523840 (see 'F')
# - Last sector: 121102335 ('End sector' - 'free space in sectors')
# press 'F' - list free unpartitioned space (100M of sectors left in the end)
# press 't' - change a partition type
# Partition number (1-3, default 3): 3
# Partition type (type L to list all types): fd
# press 'w' - write table to disk and exit
```

The kernel still uses the old table. The new table will be used at the next reboot.

```
# remove all USB drives except /dev/sda (disk1)
sudo reboot
```

Check your new partition table layout

```
sudo fdisk -l /dev/sda
```

Insert the second USB drive.

```
sudo fdisk -l /dev/sdb
```

We want the partition table of `sda` and `sdb` to be the same. Dump the disk layout from the USB drive (`/dev/sda`) to a sfdisk script file. Insert the other USB drive (aka 'disk '2'). Now both USB drives are recognized as `/dev/sda` and `/dev/sdb`. Copy the previously created disklayout (`/dev/sda`) to `/dev/sdb`.

```
sudo fdisk /dev/sda
# press 'O' - dump disk layout to sfdisk script file
# press 'q' - quit
sudo fdisk /dev/sdb
# press 'I' - load disk layout from `disklayout`
# press 'w' - write table to disk and exit
sudo fdisk -l /dev/sd[ab] # check if both drives
```

Copy the whole `/boot` partition from `/dev/sda1` to `/dev/sdb1`

```
sudo dd if=/dev/sda1 of=/dev/sdb1 status=progress
```

Make sure that there are no remains from previous RAID installations on `/dev/sda3` and `/dev/sdb3`.

```
sudo mdadm --zero-superblock /dev/sda3
sudo mdadm --zero-superblock /dev/sdb3
# mdadm: Unrecognised md component device
```
Create RAID-1 volume.

```
sudo mdadm --create /dev/md0 --level=mirror --raid-devices=2 /dev/sda3 /dev/sdb3
# Continue creating array? yes
lsblk
```

Format the RAID volume

```
sudo mkfs.ext4 /dev/md0
```

Create a mount point and mount the drive.

````
sudo mkdir -p /mnt/koekoek && sudo mount /dev/md0 /mnt/koekoek
````

Mount volume on boot

```
echo "" | sudo tee -a /etc/fstab
echo "# Koekoek RAID-1" | sudo tee -a /etc/fstab
echo "# <device>  <dir>               <type>  <options>         <dump>  <fsck>" | sudo tee -a /etc/fstab
echo "/dev/md0    /mnt/koekoek/   ext4    defaults,noatime  0       1" | sudo tee -a /etc/fstab
```

Add the new RAID drive to de mdadm configuration

```
echo "" | sudo tee -a /etc/mdadm/mdadm.conf
echo "# Koekoek RAID-1" | sudo tee -a /etc/mdadm/mdadm.conf
sudo mdadm --detail --scan |sudo tee -a /etc/mdadm/mdadm.conf
tail /etc/mdadm/mdadm.conf
```

Reboot

```
sudo reboot
```

Check the automount

```
df -h /mnt/koekoek
```

Check the RAID

```
cat /proc/mdstat
```

TODO: sending email on error does not yet work.

*I had to add the second drive again because it failed*

```
mdadm --add /dev/md0 /dev/sdb
```

#### Move system dirs to other partition

*this might fail some services*

I prefer system dirs on my RAID system. [A list of all system dirs and there function](https://thepihut.com/blogs/raspberry-pi-tutorials/34243716-what-is-actually-on-your-raspberry-pi-sd-card).

First copy the data.

```
sudo cp -av /bin /mnt/koekoek/    &&\
sudo cp -av /sbin /mnt/koekoek/   &&\
sudo cp -av /etc /mnt/koekoek/    &&\
sudo cp -av /home /mnt/koekoek/   &&\
sudo cp -av /lib /mnt/koekoek/    &&\
sudo cp -av /opt /mnt/koekoek/    &&\
sudo cp -av /tmp /mnt/koekoek/    &&\
sudo cp -av /usr /mnt/koekoek/
```

Create some new directories if needed.

```
sudo mkdir -pv /mnt/koekoek/downloads /downloads
```

Test the mount

```
sudo mount --bind /mnt/koekoek/bin  /bin  &&\ 
sudo mount --bind /mnt/koekoek/sbin /sbin &&\
sudo mount --bind /mnt/koekoek/etc  /etc  &&\
sudo mount --bind /mnt/koekoek/home /home &&\
sudo mount --bind /mnt/koekoek/lib  /lib  &&\
sudo mount --bind /mnt/koekoek/opt  /opt  &&\
sudo mount --bind /mnt/koekoek/tmp  /tmp  &&\
sudo mount --bind /mnt/koekoek/usr  /usr  &&\
sudo mount --bind /mnt/koekoek/downloads  /downloads
```

Create a mountpoint in `fstab` to do this after each boot. No worries about the directory already being there, the mount will disable them from view.

```
sudo vim /etc/fstab
# System directories are on the RAID
# /mnt/koekoek/bin  /bin    none  defaults,bind   0   0 
# /mnt/koekoek/sbin /sbin   none  defaults,bind   0   0 
# /mnt/koekoek/etc  /etc    none  defaults,bind   0   0 
# /mnt/koekoek/home /home   none  defaults,bind   0   0 
# /mnt/koekoek/lib  /lib    none  defaults,bind   0   0 
# /mnt/koekoek/opt  /opt    none  defaults,bind   0   0 
# /mnt/koekoek/tmp  /tmp    none  defaults,bind   0   0 
# /mnt/koekoek/usr  /usr    none  defaults,bind   0   0 
# /mnt/koekoek/downloads  /downloads    none  defaults,bind   0   0 
```

Reboot

```
sudo reboot
```

#### Install secure webserver/'LAMP' (MySQL, nginx, php7-fpm)

**MySQL or MariaDB**

Create the directory where the mysql socket goes by default

```
sudo mkdir -p /var/run/mysqld
```

MySQL install, here you also set the root password

```
sudo apt-get -y install mysql-server mysql-client
```

MySQL install setup (remove anonymous users etc...), defaults should be fine

```
sudo mysql_secure_installation
```

Start the service.

```
sudo service mysql restart && service mysql status
```

**Php7.0-fpm.** 

Install `php7.0-fpm` and atributes.

```
sudo apt-get install -y php7.0 php7.0-curl php7.0-mysql php7.0-gd php7.0-fpm php7.0-cli php7.0-opcache php7.0-mbstring php7.0-xml php7.0-zip
```

Setup the PHP7.0-fpm pool to use the `server` user and group.

*31-12-2017 nog niet gedaan*

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
sudo service php7.0-fpm restart && service php7.0-fpm status
command -v php && php -v
```

**Encryption**
We try to get strong encryption on our connections. Also see: *https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html*
We need generate a stronger DHE parameter.

```
# Generate a stronger DHE parameter which we use for nginx
# ! This is going to take a long time (>6 hours)
cd /etc/ssl/certs && sudo openssl dhparam -out dhparam.pem 4096
```

Certbot is an easy-to-use automatic client that fetches and deploys 'Let’s Encrypt' SSL/TLS certificates for your webserver. We need to install this from source.

```
# as 'normal user' download cert-bot in ~/bin
mkdir -p ~/bin && cd ~/bin && wget https://dl.eff.org/certbot-auto && chmod a+x certbot-auto
```

Create some certificates.

```
# run certbot in automated mode, this will install a bunch and start the setup, answers like:
~/bin/certbot-auto --standalone -d doma.in --email b.g.l.nelissen@gmail.com certonly
```

```
	# IMPORTANT NOTES:
	#  - Congratulations! Your certificate and chain have been saved at:
	#    /etc/letsencrypt/live/doma.in/fullchain.pem
	#    Your key file has been saved at:
	#    /etc/letsencrypt/live/doma.in/privkey.pem
	#    Your cert will expire on 2018-03-08. To obtain a new or tweaked
	#    version of this certificate in the future, simply run certbot-auto
	#    again. To non-interactively renew *all* of your certificates, run
	#    "certbot-auto renew"
	#  - Your account credentials have been saved in your Certbot
	#    configuration directory at /etc/letsencrypt. You should make a
	#    secure backup of this folder now. This configuration directory will
	#    also contain certificates and private keys obtained by Certbot so
	#    making regular backups of this folder is ideal.
	#  - If you like Certbot, please consider supporting our work by:
	# 
	#    Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
	#    Donating to EFF:                    https://eff.org/donate-le
```

TLDR. In the future you can renew certificates doing:
`sudo service nginx stop && sudo certbot-auto renew && sudo service nginx restart`

Todo: add line to cron for automated updates.

**Nginx**. Install the latest nginx.

```
sudo apt-get install -y nginx
```

Now we need to create the root directories `www` and `port8080`.

```
# create web directory (asif we are 'server')
sudo -u server mkdir /home/server/www && ls -la  /home/server/www
sudo -u server mkdir /home/server/port8080 && ls -la  /home/server/port8080
```

```
# create test index files
echo "test <b>www</b><p><?php phpinfo(); ?>" | sudo -u server tee -a /home/server/www/index.html
echo "test <b>port8080</b><p><?php phpinfo(); ?>" | sudo -u server tee -a /home/server/port8080/index.html
```

Replace the contents of /etc/nginx/nginx.conf 

```
#  # http://nginx.org/en/docs/ngx_core_module.html
#  # b.nelissen
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
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.original && \
sudo vim /etc/nginx/nginx.conf
```

The server configuration file

```
#  # doma.in
#  server {
#    server_name doma.in;
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
#    ssl_certificate /etc/letsencrypt/live/doma.in/fullchain.pem;
#    ssl_certificate_key /etc/letsencrypt/live/doma.in/privkey.pem;
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
sudo vim /etc/nginx/sites-available/doma.in.conf
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
#    index index.php index.html;
#  }
sudo vim /etc/nginx/sites-available/port8080.conf
```

Redirect all port 80 trafic to port 443

```
#  # redirect port 80 to secure 443 ssl
#  server {
#    listen 80;
#    server_name doma.in;  
#    return 301 https://$server_name$request_uri;
#  }
sudo vim /etc/nginx/sites-available/redirect80to443.conf
```

Remove the default configuration

```
sudo rm /etc/nginx/sites-enabled/default
```

Link all the configurations files from the 'available' directory to the 'enabled' directory

```
cd /etc/nginx/sites-enabled/
sudo ln -s ../sites-available/doma.in.conf ./
sudo ln -s ../sites-available/port8080.conf ./
sudo ln -s ../sites-available/redirect80to443.conf ./
ls /etc/nginx/sites-enabled
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

**Check SSL security** https://www.ssllabs.com/ssltest/analyze.html?d=doma.in&latest

#### Mount external USB drive in `fstab`

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
# https://download.owncloud.org/community/owncloud-10.0.4.tar.bz2
# 
cd ~/ && wget https://download.owncloud.org/community/owncloud-10.0.4.tar.bz2

# extract
tar jxfv owncloud-10.0.4.tar.bz2

# move 'owncloud' to www
mv owncloud ~/www/
```

If needed you can move the data directory to another drive.

```
# create a 'data' directory on the usb drive
sudo -u server mkdir /home/server/owncloud-data
```

Prepare MySQL with a user and a database

```
# go back to user 'bas' which is able to do a sudo
# login mysql as root and create new database + user
sudo mysql -uroot -p
#   CREATE DATABASE IF NOT EXISTS owncloud;
#   CREATE USER 'koekoek'@'localhost' IDENTIFIED BY 'Pa$$W0rd';
#   GRANT ALL PRIVILEGES ON owncloud.* TO 'koekoek'@'localhost' IDENTIFIED BY 'Pa$$W0rd';
#   FLUSH PRIVILEGES;
#   quit
```

Setup ownCloud web-configuration and ownCloud should be up and running.

```
# ownCloud setup via the browser on your Mac
# username [koekoek]
# password [create admin password]
# data [/home/server/owncloud-data]
```

When your owncloud data is not empty, you want to rescan all your files.

```
cd /home/server/www/owncloud
./occ files:scan --all
./occ files:clean
```

Install the `Text Editor`. 
Login as server

```
# login as 'server'
sudo su server
```

Preferably install apps using the Market place which is available in the admin account.

- Text Editor


Download the `Notes` app for ownCloud.

```
wget https://github.com/owncloud/notes/archive/master.zip -O ~/files_notes.zip
unzip ~/files_notes.zip && mv -v ~/notes-master ~/www/owncloud/apps/notes
```

Enable `Notes` app in ownCloud.

```
cd ~/www/owncloud/ && ./occ app:enable notes
cd ~/www/owncloud/ && ./occ integrity:check-app notes
```


#### phpMyAdmin

Download phpMyAdmin and add it to 'www'

```
# login as 'server'
sudo su server

# Check https://www.phpmyadmin.net/downloads/ for the latest and greatest
# https://files.phpmyadmin.net/phpMyAdmin/4.7.6/phpMyAdmin-4.7.6-english.tar.gz
PHPVERSION="phpMyAdmin-4.7.7-english"
# https://files.phpmyadmin.net/phpMyAdmin/4.7.7/phpMyAdmin-4.7.7-english.tar.gz
cd ~/ && wget https://files.phpmyadmin.net/phpMyAdmin/4.7.7/"$PHPVERSION".tar.gz

# extract and delete tar
tar xzfv "$PHPVERSION".tar.gz && rm "$PHPVERSION".tar.gz

# move 'phpMyAdmin' to 'www'
mv "$PHPVERSION" -v ~/www/phpmyadmin

# create a cookie blowfish secret
cd ~/www/phpmyadmin/ && cp config.sample.inc.php config.inc.php

# edit the blowfish_secret in config.inc.php
#   $cfg['blowfish_secret'] = 'superrandomgibrish@$#^%SDFGASDF@#$';
vim config.inc.php

# back to user `bas`
exit
```

Setup phpMyAdmin web-configuration (https://doma.in/phpmyadmin), login with root.

Create a new database user named server

1. Go to home
2. User accounts
3. 'Add user account'
4. User name 'server', Host name 'localhost', Password

#### OpenVPN server using piVPN

Install dependencies (which the script does not check...)

```
sudo apt-get -y install openvpn easy-rsa git iptables-persistent dnsutils expect unattended-upgrades
# save ip rules when asked.
```

Run the PiVPN script

```
# run install script as normal user
curl -L https://install.pivpn.io | bash
# static address?: yes
# local user: bas
# enable unattended upgrades: yes
# protocol UDP
# port 1194
# 4096 bit prime should be enough
# Use DNS and enter the domain
# Choose google as a DNS server
# reboot
# Now run 'pivpn add' to create the ovpn profiles.
# Run 'pivpn help' to see what else you can do!
# The install log is in /etc/pivpn.
```

```
# is openvpn running?
service openvpn status
```

#### Git server

Install git

```
sudo apt-get -y update && sudo apt-get install -y wget git-core
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
git remote add KoekoekPi bas@doma.in:/home/bas/git/bin.git
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
sudo apt-get -y update && sudo apt-get -y install transmission transmission-cli transmission-daemon
```

Create download directories

```
sudo mkdir -pv /downloads/Torrent/Complete /downloads/Torrent/Incomplete
```

Stop transmission-daemon before changing settings

```
sudo service transmission-daemon stop
```

Edit the settings file:

```
sudo vim /etc/transmission-daemon/settings.json
# change tmp/download dir in settings file
#   "blocklist-enabled": true, 
#   "blocklist-url": "http://list.iblocklist.com/?list=bt_level1",
#   "download-dir": "/downloads/Torrent/Complete",
#   "incomplete-dir": "/downloads/Torrent/Incomplete",
#   "incomplete-dir-enabled": true,
#   "rpc-authentication-required": false,
#   "rpc-whitelist": "127.0.0.1,10.0.*.*,192.168.*.*",
```

Transmission runs as user debian-transmission, and in group debian-transmission. Add the local user to that group and change the permissions of the Torrent directory

```
sudo usermod -a -G debian-transmission bas
sudo chown --recursive debian-transmission:debian-transmission /downloads/Torrent
```

Start me up!

```
sudo service transmission-daemon restart
service transmission-daemon status
```

The Transmission web-interface can be found at: [10.0.0.100:9091](10.0.0.100:9091).
- Click the wrench, Peers, 'Enable' blocklist and 'update'
- Don't forget to open the port in the router (e.g. '51413' TCP)

#### Citadel: Mail, Address book & Calendar server

Citadel is pre-configured so that IPv4 and IPv6 are set as the default transfer protocols.

```
# activate ipv6
sudo modprobe ipv6
```

Install Citadel

```
sudo apt-get -y install libldb-dev libldap2-dev citadel-suite 
# Listening address for the Citadel server: 0.0.0.0 *default*
# Authentication method to use: Internal
# Integration with Apache webservers: Internal
# Webcit HTTP port: 2000
# Webcit HTTPS port: 2001
# Limit Webcit's login language selection: User-defined
# ...
service citadel status
```

When receiving `citserver[10040]: configuration setting c_default_cal_zone is empty, but must not - check your config!`

```
# notice there is no `c_default_cal_zone` variable set
sudo sendcommand conf listval

# set the variable
sudo sendcommand "CONF PUTVAL|c_default_cal_zone|timezone"

# now the variable is set.
sudo sendcommand conf listval
```

Continue installation using the citadel install script

```
cd ~
curl http://easyinstall.citadel.org/install > citadel-install.sh
sudo bash ./citadel-install.sh
# takes a long (>1hour) time
# yes, yes
# Choose IMAP for receiving, and SMTP for sending.

# - Citadel will be installed to `/usr/local/citadel`
# - WebCit will be installed to `/usr/local/webcit`
# - supporting libraries will be installed to `/usr/local/ctdlsupport`
# - The `setup` program will initialize (/usr/local/citadel/setup)
# - http://www.citadel.org/doku.php/installation:getting_started
```

Change the default ports to acces Citadel web environment away from 80 and 443, and reboot

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

Setup your admin user.

```
sudo dpkg-reconfigure citadel-server
```


The rest of the setup goes through the web interface.
Point your web browser at https://10.0.0.100:2000


- Administration
  - Edit site-wide configuration
	- General:
	  - Fully qualified domain name: `doma.in`
	  - Geographic location of this system: `Netherlands`
	- Settings:
	  - Server IP address: `*`
	  - XMPP: `-1`
	- POP3
	  - POP3 listener port: `-1`
	  - POP3 over SLL port: `-1`
  - Domain names and Internet mail configuration
	- Local host aliases: `doma.in` add
  - Add, change, delete user accounts
	- Add user
  - Restart now

When you login as a user you can setup forward rules

- Advanced
  - View/edit server-side mail filters
	- `if All forward to: example@mail.com`

Open ports in your router. SMTP is used for relaying messages between client/server, open port 25 and 465 TCP in the firewall/router. IMAP for accessing mail via your client, open port 993 TCP in the firewall/router.

Setup for the iPhone:

1. Settings, Mail, Accounts, Add Account, Other...
2. Add Mail account
3. Name, name@doma.in, password, doma.in mail
3. Incoming mail server
  - Host name: doma.in
  - Username: name
  - Password: password
4. Outgoing mail server
  - Host name: doma.in
  - Username: name
  - Password: password
5. Verifying account takes a few minutes, patience.
6. Test your settings in Mail, you should be able to send/receive email from doma.in

#### Wondershaper

Wonder Shaper uses `iproute` to *shape* traffic and limit the up and download speeds. My experience is that full bandwith upload traffic to the pi can choke my whole LAN. Therefor I throttle my Pi to keep the LAN from slowing down.

```
sudo apt-get -y install wondershaper speedtest-cli
```

Check your interfaces with `ifconfig`, eg. eth0.

Test an appropriate setting, like down 12000kb/s and up 1600kb/s

```
# test the speed befor throttle

# sudo wondershaper eth1 downspeed upspeed
sudo wondershaper eth0 12000 1600
# to undo wondershaper run
# sudo wondershaper clear eth0

# test the speed after throttle
curl -s  https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py | python -

```

Set the upload and download speeds over a reboot. The following makes shure the settings apply when the network interface comes up, and the settings are cleaned when the network interface goes down.

```
# backup current network settings
sudo cp -v /etc/network/interfaces /root/interfaces."$(date +%Y%m%d%H%M)".bak
# add the following to /etc/network/interfaces under eth0
# up /sbin/wondershaper eth0 12000 1600
# down /sbin/wondershaper clear eth0
sudo -s vim /etc/network/interfaces
```

#### DenyHosts script

DenyHosts is a script intended to be run by Linux system administrators to help thwart SSH server attacks (also known as dictionary based attacks and brute force attacks).
When run for the first time, DenyHosts will create a work directory. The work directory will ultimately store the data collected and the files are in a human readable format, for each editing, if necessary.
DenyHosts then processes the sshd server log (typically, this is /var/log/secure, /var/log/auth.log, etc) and determines which hosts have unsuccessfully attempted to gain access to the ssh server. Additionally, it notes the user and whether or not that user is root, otherwise valid (eg. has a system account) or invalid (eg. does not have a system account).

When DenyHosts determines that a given host has attempted to login using a non-existent user account a configurable number of attempts, DenyHosts will add that host to the /etc/hosts.deny file. This will prevent that host from contacting your sshd server again.

Also, DenyHosts will note any successful logins that occurred by a host that has exceeded the deny_threshold. These are known as suspicious logins and should be investigated further by the system admin.

```
sudo apt-get -y install denyhosts geoip-bin
```

To see if DenyHosts is running. While denyhosts is running it will update the `/etc/hosts.deny` file. DenyHosts runs in the background.

```
ps -ax | grep [d]enyhosts
```

Parse the `/etc/hosts.deny` file to see where these lame scriptkiddies come from

```
for i in $(tail -n +17 /etc/hosts.deny | awk '{print $2}');do printf "$i "; geoiplookup $i;done
```

You can set `/etc/hosts.allow` to always allow local login attempts:

```
# add the following lines, the last line must be blanc!
# ALL : 192.168.,10.0.
# 

sudo vim /etc/hosts.allow
```

#### Setup Mosquitto, the MQTT host

Install using apt-get

```
sudo apt-get install -y mosquitto mosquitto-clients
```

Test the mosquitto server using 2 console windows.

Window 1

```
# start server
mosquitto_sub -t test-topic_mqtt -d
```

Window 2

```
mosquitto_pub -d -t test-topic_mqtt -m "Sebe is homo"
```

Window 1 should see the message of Window 2.

#### Node-red

Update system and install using the `node-red` installer script.

```
sudo apt-get install -y build-essential 
bash <(curl -sL https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/update-nodejs-and-nodered)
```

Autostart `node-red` on boot and reboot to finish installation.

```
sudo systemctl enable nodered.service
sudo reboot
```

Go to http://koekoek:1880 to use node-red.


## HomeKit

Homekit, todo



## Extra's

#### Original cmdline.txt and config.txt contents

```
cat /boot/cmdline.txt 
# dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=PARTUUID=b184bfd6-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait
```

```
cat /boot/config.txt 
# For more options and information see
# http://rpf.io/configtxt
# Some settings may impact device functionality. See link above for details
# 
# The whole file is commented out, so not functionally empty
```

#### Blinking red LED

A blinking red LED means insufficient power to the device. This can cause SD card failure or other weird IO bugs.

#### Disk/resource usage

```
sudo iotop --only
```

#### Disk failures

Diskfailures do happen. For a clean guide check https://www.cyberciti.biz/tips/surviving-a-linux-filesystem-failures.html

```
# force checking of a drive
sudo e2fsck
```

#### Network monitoring tools

Install the tools you want:

```
sudo apt-get install -y \
iotop \
nload \
iftop \
iptraf \
nethogs \
bmon \
slurm \
tcptrack \
vnstat \
darkstat \
cbm \
geoip-bin
```

**Darkstat**

For longer term monitoring I would suggest something like `darkstat`. Darkstat continuesly monitors network traffic and shows it on a website. You can run it once, or as a service. 

- once: `darkstat -i eth0 -p 82`, go to koekoekpi.local:82 to get the interface.
- at boot:

```
# START_DARKSTAT=yes
# INTERFACE="-i eth0"
# PORT="-p 82"
sudo vim /etc/darkstat/init.cfg

# start her up!
sudo service darkstat restart
sudo service darkstat status
```

**iftop**

For immediate monitoring you can use `iftop`. This will show you the currently active connections and the bandwidth they are using. Once you've identified a high traffic connection, find the local port number and use netstat to find which process the connection belongs to.

`iftop`, press t, then shift P. Copy the ip:port and to find the application run:

```
netstat -tnp | grep 10.0.0.100:51413
```

**geo-ip**

Parse the `/etc/hosts.deny` file to see where these lame scriptkiddies come from

```
for i in $(tail -n +17 /etc/hosts.deny | awk '{print $2}');do printf "$i "; geoiplookup $i;done
```

## Backup workflow

#### Update certificates

```
# stop webserver to free the port for certbot
sudo service nginx stop

# run certbot in automated mode
~/bin/certbot-auto --standalone -d doma.in --email b.g.l.nelissen@gmail.com certonly

# start the webserver
sudo service nginx start
```

#### Create backup image of the current system

ToDo: clean trash and logs befor backup:

```
# remove all archived logs and numbered logs
# why not remove all of them?
# rm /var/log/*
# rm /var/log/*.gz
# rm /var/log/*.1
```

```
# clean downloaded apt-get packages
sudo apt-get clean
```

Shutdown the pi gracefully

```
sudo shutdown -t now
```

#### Create backup from SD/USB

```
sudo date && diskutil list && \
read -p "Enter the disk NUMBER you want to backup and press [ENTER]: " DISK && \
diskutil unmountDisk /dev/disk${DISK} && \
sudo pv /dev/rdisk${DISK} | pigz -9 > KoekoekPi."$(date +%Y%m%d)".backup.img.gz
```

#### Restore `.backup.img.gz` backup to SD/USB

**After restore, your ownCloud is likely to sync (delete) to the previous date! Do a '`occ files:scan`' first (see 'OwnCloud' instructions).**

```
IMGGZ="KoekoekPi.20171016.backup.img.gz";
sudo date && diskutil list && \
read -p "Enter the disk NUMBER you want to restore from and press [ENTER]: " DISK && \
diskutil unmountDisk /dev/disk${DISK} && \
gzip --decompress --stdout ${IMGGZ} | pv -s "$(du -s -B1 "$IMGGZ" | cut -f 1)" | sudo dd of=/dev/rdisk${DISK} bs=100M && \
diskutil unmountDisk /dev/disk${DISK}
```
