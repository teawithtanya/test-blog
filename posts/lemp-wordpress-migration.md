---
title: LEMP WordPress Migration
description: I manually migrated a LEMP WordPress site and took some notes along the way.
date: 2022-07-11
tags:
  - LEMP
  - WordPress
layout: layouts/post.njk
---

I manually migrated an old LEMP WordPress site and took some notes along the way.

## Old Server

Linux, Nginx, MySql, PHP versions

```bash
$ cat /etc/system-release
Amazon Linux AMI release 2011.09

$ nginx -v
nginx version: nginx/0.8.54

$ mysql -V
mysql  Ver 14.14 Distrib 5.1.61, for redhat-linux-gnu (i386) using readline 5.1

$ php -v
PHP 5.3.13 (cli) (built: May  9 2012 18:39:36)
Copyright (c) 1997-2012 The PHP Group
Zend Engine v2.3.0, Copyright (c) 1998-2012 Zend Technologies

```

WordPress version: `3.2.1`

## New EC2 Instance

Launched new EC2 Instance with the following options:

*  Amazon Linux 2 Kernel 5.10 AMI 2.0.20220606.1 x86_64 HVM gp2
   ami-098e42ae54c764c35

*  t2.micro
*  1 volume(s) - 8 GiB

## Migrated data on EBS volume

The old server had WordPress content and MySql data on a separate EBS volume.
Created snapshot of this data volume of old EC2 Instance.
Created a new volume from snapshot - 5 GiB, in the same Availability Zone.
Attached the data volume to new EC2 instance as device `/dev/xvdf`.

```bash
$ sudo fdisk -l
Disk /dev/xvda: 8 GiB, 8589934592 bytes, 16777216 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
(**redacted**)

Disk /dev/xvdf: 5 GiB, 5368709120 bytes, 10485760 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

Mounted data volume

```bash
$ sudo mkdir -p /ebs/data
$ sudo mount /dev/xvdf /ebs/data
$ df -h | grep xvd
/dev/xvda1      8.0G  1.6G  6.5G  20% /
/dev/xvdf       5.0G  4.0G  1.1G  80% /ebs/data
```

Fixed permissions

```bash
$ sudo chown -R ec2-user.ec2-user /ebs/data/www
```

## Installed Nginx, MySql, PHP

```bash
$ sudo amazon-linux-extras install nginx1

$ sudo yum install mysql-server mysql-client
Loaded plugins: extras_suggestions, langpacks, priorities, update-motd
amzn2-core                                                                                                                                                                                                                 | 3.7 kB  00:00:00
No package mysql-server available.
No package mysql-client available.
Error: Nothing to do

```

Hmmm...

According to [AWS Tutorial: Install a LAMP web server on Amazon Linux 2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-lamp-amazon-linux-2.html), AWS recommends MariaDB for Amazon Linux 2.

```bash
$ sudo yum install mariadb-server
$ sudo amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
```
Went with AWS recommended PHP 7.2 (instead of the latest 7.4). AWS says security fixes have been backported to this version. But applications may still show a warning.

## Customized Nginx

The following settings are aimed to run Nginx with limited resources (e.g., t2.micro EC2 instance). Edited `/etc/nginx/nginx.conf`

```bash
# note: output partially truncated for brevity
$ cat /etc/nginx/nginx.conf

# changed worker_processes to 1
worker_processes 1;

http {

    # enabled these
    gzip on;
    gzip_disable "msie6";
    gzip_proxied any;
    gzip_comp_level 3; # 1 is least compression, 9 is most
                       # kept this low due to limited CPU
    gzip_buffers 16 8k;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    # The default server
    server {
      listen       80;
      listen       [::]:80;
      server_name  _;

      access_log /ebs/data/www/logs/access.log;
      error_log /ebs/data/www/logs/error.log;
      root /ebs/data/www/htdocs;

      # Load configuration files for the default server block.
      include /etc/nginx/default.d/*.conf;

      error_page 404 /404.html;
      location = /404.html {
      }

      error_page 500 502 503 504 /50x.html;
      location = /50x.html {
      }

      location /legacy {
          rewrite ^/legacy/?$ /blog/ permanent;
          rewrite ^/legacy/index\.html$ /blog/ permanent;
          rewrite ^/legacy/archives/00([0-9]*)\.html$ /blog/mt-redirect.php?postid=$1 permanent;
      }

      if (!-e $request_filename) {
          rewrite ^(.+)$ /blog/index.php?q=$1 last;
      }

    }
}

$ cat /etc/nginx/default.d/php.conf
index index.php index.html index.htm;

location ~ \.(php|phar)(/.*)?$ {
    fastcgi_split_path_info ^(.+\.(?:php|phar))(/.*)$;

    fastcgi_intercept_errors on;
    fastcgi_index  index.php;
    include        fastcgi_params;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    fastcgi_param  PATH_INFO $fastcgi_path_info;
    fastcgi_pass   php-fpm;
}

# Backed up nginx settings to data volume
$ sudo cp -r /etc/nginx/ /ebs/data/etc/

```
## Customized MySQL

```bash
# Disabled MySQL InnoDB and Enable MyISAM
$ cat /etc/my.cnf
# note: output partially truncated for brevity
[mysqld]
skip-innodb
default-storage-engine = myisam

$ sudo service mariadb restart
```
To import a MySql backup of WordPress, I previously created an empty database first.

```bash
$ mysqladmin -uroot password *****
$ mysqladmin -u root -p create "myblogdb"
$ echo "GRANT ALL PRIVILEGES ON myblogdb.* TO 'blogadmin'@'%' IDENTIFIED BY '*****';" | mysql -u root -p
$ echo "GRANT ALL PRIVILEGES ON myblogdb.* TO 'blogadmin'@'locahost' IDENTIFIED BY '*****';" | mysql -u root -p

# MySql backup can be imported
```

Alternately, I moved MySql to /ebs

```bash
$ sudo systemctl stop mariadb
$ sudo mv /var/lib/mysql /var/lib/mysql_old
$ sudo mkdir /var/lib/mysql
$ sudo mount --bind /ebs/data/lib/mysql /var/lib/mysql -o noatime
$ sudo systemctl start mariadb
```

## Customized PHP, PHP-FPM

```bash
$ sudo chown -R ec2-user.ec2-user /var/log/php-fpm

$ cat /etc/php.ini
# Set default timezone
date.timezone = "America/Los_Angeles"

$ timedatectl list-timezones | grep Los
America/Los_Angeles
$ sudo timedatectl set-timezone America/Los_Angeles
$ timedatectl status

$ grep -v "^;" /etc/php-fpm.d/www.conf | awk NF
[www]
user = ec2-user
group = ec2-user
listen = /run/php-fpm/www.sock
listen.acl_users = apache,nginx
listen.allowed_clients = 127.0.0.1
pm = dynamic
pm.max_children = 5
pm.start_servers = 1
pm.min_spare_servers = 1
pm.max_spare_servers = 2
slowlog = /var/log/php-fpm/www-slow.log
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on
php_value[session.save_handler] = files
php_value[session.save_path]    = /var/lib/php/session
php_value[soap.wsdl_cache_dir]  = /var/lib/php/wsdlcache

# Backed up PHP-FPM settings to data volume
$ sudo cp -r /etc/php-fpm.d/ /ebs/data/etc/

# Installed PHP Opcode caching
$ sudo yum install php-opcache

# Started Nginx and PHP-FPM.

$ sudo systemctl start nginx
$ sudo systemctl start php-fpm

```

## Customized server startup

```bash
# Ensured MySql, php-fpm, Nginx does NOT start automatically
$ systemctl list-unit-files | grep mariadb
mariadb.service                               disabled
mariadb@.service                              disabled
$ systemctl list-unit-files | grep nginx
nginx.service                                 disabled
$ systemctl list-unit-files | grep php-fpm
php-fpm.service                               disabled

# Custom start up script
$ sudo mkdir /etc/serverclevercat
$ sudo vi /etc/serverclevercat/S99ebs-mounts.sh
$ cat /etc/serverclevercat/S99ebs-mounts.sh
#!/bin/bash
# Script checks to see if EBS is attached at /dev/xvdf
# then mounts and starts services
if [ -b /dev/xvdf ]; then
  mount /dev/xvdf /ebs/data
  mount --bind /ebs/data/lib/mysql /var/lib/mysql -o noatime
  systemctl start mariadb
  systemctl start php-fpm
  systemctl start nginx
fi

$ sudo chmod u+x /etc/serverclevercat/S99ebs-mounts.sh

# Enabled script at startup
$ sudo vi /etc/systemd/system/ebs-mounts.service
$ cat /etc/systemd/system/ebs-mounts.service
[Unit]
Description=Run script at startup
After=default.target

[Service]
Type=simple
RemainAfterExit=yes
ExecStart=/etc/serverclevercat/S99ebs-mounts.sh
TimeoutStartSec=0

[Install]
WantedBy=default.target

$ sudo systemctl daemon-reload
$ sudo systemctl enable ebs-mounts
Created symlink from /etc/systemd/system/default.target.wants/ebs-mounts.service to /etc/systemd/system/ebs-mounts.service.
$ systemctl list-unit-files | grep ebs-mounts
ebs-mounts.service                            enabled

```

## Testing

Tested website. Not working. :(

Error message in logs looked like

```bash
PHP message: PHP Fatal error:  Uncaught Error: Call to undefined function set_magic_quotes_runtime()
```

Likely WordPress is too old for latest PHP.

Used guide to update WordPress manually: https://thrivewp.com/update-wordpress-manually

```bash
$ mkdir download
$ cd download
$ wget https://wordpress.org/latest.tar.gz

# Delete the old 'wp-includes' and 'wp-admin' directories, and replace with new versions
$ cd ../..
$ mkdir old
$ mv blog/wp-includes old/
$ mv blog/wp-admin old/
$ mv download/wordpress/wp-includes blog/
$ mv download/wordpress/wp-admin blog/

# Overwrote old wp-content, root files with new versions (taking care not to delete all old content)
$ cp -r download/wordpress/wp-content/* blog/wp-content/
$ cp download/wordpress/* blog/

```

Checked nginx error_log for some PHP errors and fixed them
Disabled 50x error page in nginx.conf so the website will display PHP errors

```bash
#error_page 500 502 503 504 /50x.html;
        #location = /50x.html {
        #}

```
This also helped fix a few broken plugins

```bash
# Deleted some old plugins that weren's working
$ cd blog/wp-content
$ rm -r plugins/w3-total-cache
$ rm w3-total-cache-confid.php
$ rm -r w3tc
$ rm db.php
$ rm object-cache.php

```
