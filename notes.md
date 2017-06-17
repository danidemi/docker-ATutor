atutor
======

docker php jpeg --with-jpeg-dir=/usr/lib

https://sourceforge.net/projects/atutor/files/latest/download
https://sourceforge.net/projects/atutor/files/ATutor%202/ATutor-2.2.2.tar.gz/download
https://downloads.sourceforge.net/project/atutor/ATutor%202/ATutor-2.2.2.tar.gz?r=https%3A%2F%2Fsourceforge.net%2Fprojects%2Fatutor%2Ffiles%2FATutor%25202%2F&ts=1497367337&use_mirror=netix


ATutor is a php webapp, so it should probably run in a php enabled server.

We have a 7.1.6-apache official php image, let's try it.

docker run php:7.1.6-apache


docker exec -it "id of running container"
/var/www/html here is the web site

docker run -p 80:80 -v /home/user/Worspace/atutor/ATutor:/var/www/html php:7.1.6-apache 

lot of errors of this type

===========
Deprecated: Methods with the same name as their class will not be constructors in a future version of PHP; Message has a deprecated constructor in /var/www/html/include/classes/Message/Message.class.php on line 20

Warning: Cannot modify header information - headers already sent by (output started at /var/www/html/include/classes/Message/Message.class.php:20) in /var/www/html/install/index.php on line 22

Warning: Cannot modify header information - headers already sent by (output started at /var/www/html/include/classes/Message/Message.class.php:20) in /var/www/html/install/index.php on line 23
===========

things that are not accepted by ATutor:
- mysql DISABLED
- GD     Disabled
- session.save_path     Directory Not Writeable
- MySQL 4.1.10+     Not Found    Bad
- Javascript Enabled?     Disabled    Bad


So, it seems that mysql_x functions disappeared from php7.
We should probably use php 5.6.30!


docker run -p 80:80 -v /home/user/Worspace/atutor/ATutor:/var/www/html php:5.6.30-apache 

not yet, inside the container you should run

docker-php-ext-install mysql

YES! mysql enabled!

Now GD is down. let's try 

docker-php-ext-install gd

apt-get install libpng-dev ... umh no!

docker-php-ext-install -j5 gd

no! maybe this one

apt-get update && \
            apt-get install -y libfreetype6-dev libjpeg62-turbo-dev && \
            docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/  &&  \
            docker-php-ext-install gd


yes! all green. Now I suppose the installation will break



ok, let's suppose that is the mysql stuff that is not working
let's start a contianer

docker run -e MYSQL_ROOT_PASSWORD=rootpwd mysql:5.7.18

docker container inspect <mysql container> to discover the IP

then atutor wants 


mkdir -p /var/content
chmod 2777 /var/content


then atutor needs 

chmod a+w -R include/

on the include where the content of atutor is

it probably wrote this... ../include/config.inc.php


Now errors on timezones....

==========================================

Warning: date(): It is not safe to rely on the system's timezone settings. 
You are *required* to use the date.timezone setting or the date_default_timezone_set() function. 
In case you used any of those methods and you are still getting this warning, you most likely misspelled the timezone identifier. 
We selected the timezone 'UTC' for now, but please set date.timezone to select your timezone. in /var/www/html/include/lib/mysql_connect.inc.php on line 205

Warning: getdate(): It is not safe to rely on the system's timezone settings. You are *required* to use the date.timezone setting or the date_default_timezone_set() function. In case you used any of those methods and you are still getting this warning, you most likely misspelled the timezone identifier. We selected the timezone 'UTC' for now, but please set date.timezone to select your timezone. in /var/www/html/include/classes/ErrorHandler/ErrorHandler.class.php on line 508

Warning: getdate(): It is not safe to rely on the system's timezone settings. You are *required* to use the date.timezone setting or the date_default_timezone_set() function. In case you used any of those methods and you are still getting this warning, you most likely misspelled the timezone identifier. We selected the timezone 'UTC' for now, but please set date.timezone to select your timezone. in /var/www/html/include/classes/ErrorHandler/ErrorHandler.class.php on line 301
==========================================

php.ini
date.timezone string 



let's use the...

========================
echo "<?php phpinfo(); ?>" > phpinfo.php
========================


what we get is:

=========================
Configuration File (php.ini) Path     /usr/local/etc/php
Loaded Configuration File     (none)
Scan this dir for additional .ini files     /usr/local/etc/php/conf.d
Additional .ini files parsed     /usr/local/etc/php/conf.d/docker-php-ext-gd.ini, /usr/local/etc/php/conf.d/docker-php-ext-mysql.ini 
=========================

can we probably add a php.ini here ?

echo "date.timezone=Europe/Rome" > /usr/local/etc/php/conf.d/docker-php.ini


OK, it works.
NOw a lot of errors like 

=====================
Warning: Invalid argument supplied for foreach() in /var/www/html/include/vitals.inc.php on line 374
Warning: Invalid argument supplied for foreach() in /var/www/html/include/lib/output.inc.php on line 288
Warning: Invalid argument supplied for foreach() in /var/www/html/include/lib/output.inc.php on line 288
[ the_follow_errors_occurred ]
[ AT_ERROR_DB_QUERY ]
=====================

According to http://www.atutor.ca/view/7/24670/1.html it could be an error about the collate of the table.

Why not to start a mysqladmin ?


https://hub.docker.com/r/phpmyadmin/phpmyadmin/

docker run -e MYSQL_ROOT_PASSWORD=root_pwd -d --name mysql -e PMA_HOST=172.17.0.3 -e PMA_PORT=3306 -p 8080:80 phpmyadmin/phpmyadmin


phpmyadmin starts but no clues


Noooo! Log is disabled by default. So, how can I now what is wrong with php ?
I fear I have to set up the logger


[Sat Jun 17 14:07:12.530641 2017] [:error] [pid 25] [client 172.17.0.1:36962] Table 'atutor.AT_language_text' doesn't exist, referer: http://localhost/login.php

172.17.0.3
3306
root
rootpwd
atutor
AT_

