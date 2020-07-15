##
Install and configure freeradius to use mariadb database (also for managing freeradius, we install daloRADIUS web interface).
===

1- Install and setup mariadb
To install mariadb, we create mariadb repository file and install required packages. here we install mariadb 10.5:

```
# vim /etc/yum.repos.d/mariadb.repo
```
then put the following content in it:

```
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.5/centos7-a...
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-M...
gpgcheck=1
now install mariadb:
# yum install mariadb-server mariadb-client
```

then start mariadb service:
```
# systemctl start mariadb
# systemctl enable mariadb
```

then do initial mariadb setup:
```
# mysql_secure_installation
```

now we should create a user and database for freeradius in mariadb:

Note: change “radiuspassword” with your desired password.
```
# mysql -u root -p
# CREATE DATABASE radius;
# GRANT ALL ON radius.* TO radius@localhost IDENTIFIED BY "radiuspassword";
# FLUSH PRIVILEGES;
# quit;
```

2- Install apache and php
for a managing freeradius through daloRADIUS web interface we need to install apache and php:
```
# yum install https://dl.fedoraproject.org/pub/epel...
# yum install yum-utils -y
# yum install http://rpms.remirepo.net/enterprise/r...
# yum-config-manager --enable remi-php73
# yum install php php-common php-opcache php-mcrypt php-cli php-gd php-curl php-mysqlnd php-pear php-pear-DB
```
then install apache:
```
# yum install httpd
```

3- Install and configure freeradius
```
# vim /etc/yum.repos.d/base.repo
[base]
name=CentOS-7 - Base
mirrorlist=http://mirrorlist.centos.org/?release...
#baseurl=http://mirror.centos.org/centos/7/os/...
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

# yum install freeradius freeradius-utils freeradius-mysql freeradius-perl php-pear
```

then we import freeradius schema:
```
# mysql -u root -p radius ( /etc/raddb/mods-config/sql/main/mysql/schema.sql
```

and create a soft link to available mods:
```
# ln -s /etc/raddb/mods-available/sql /etc/raddb/mods-enabled/sql
```
 
Now open etc/raddb/mods-available/sql and make change to be like the following:
```
# vim /etc/raddb/mods-available/sql
sql {
driver = "rlm_sql_mysql"
dialect = "mysql"
# Connection info:
server = "localhost"
port = 3306
login = "radius"
password = "radiuspassword"
# Database table configuration for everything except Oracle
radius_db = "radius"
}
read_clients = yes
client_table = "nas"
then open /etc/raddb/clients.conf and change ipaddr and proto to be same as the following:

ipaddr = *
proto = tcp
```

4- Install and configure daloRADIUS
Install and configure daloRADIUS. its package is available in github. so download it and extract:
```
# wget https://github.com/lirantal/daloradiu...
# unzip master.zip
# mv daloradius-master/ daloradius
# cd daloradius
```

Import daloRadius tables into database:
```
# mysql -u root -p radius ( contrib/db/fr2-mysql-daloradius-and-freeradius.sql
# mysql -u root -p radius ( contrib/db/mysql-daloradius.sql
```

Move its directory to apache root document:
```
# cd ..
# mv daloradius /var/www/html/
```

Change owner of daloRadius and set proper selinux policy:
```
# chown -R apache:apache /var/www/html/daloradius/
# chmod 664 /var/www/html/daloradius/library/daloradius.conf.php
# restorecon -R /var/www/html/daloradius/
```
Open daloRadius config file and set the following parameters:
```
# vim /var/www/html/daloradius/library/daloradius.conf.php
$configValues['CONFIG_DB_USER'] = 'radius';
$configValues['CONFIG_DB_PASS'] = 'radiuspassword';
$configValues['CONFIG_DB_NAME'] = 'radius';
```

5- Configure firewall
we need to open radius and web port. so issue these commands:
```
# firewall-cmd --permanent --add-port=1812/tcp
# firewall-cmd --permanent --add-port=1812/udp
# firewall-cmd --permanent --add-port=1813/tcp
# firewall-cmd --permanent --add-port=1813/udp
# firewall-cmd --permanent --add-port=80/tcp
```
then reload firewall:
```
# firewall-cmd --reload
```

6- Start services
In rare circumstances, selinux policy manager may be crashed when we start freeradius server. so first update some selinux packages:
```
# yum update setools checkpolicy policycoreutils
```

Freeradius and daloRadius installation and configurations has been done. last thing is to start services:
```
# systemctl start radiusd.service
# systemctl restart mariadb.service
# systemctl restart httpd
# systemctl enable radiusd.service
# systemctl enable mariadb.service
# systemctl enable httpd
```
Then in your browser, point to this address: (remember to change IP address with your own)
```
http://localhost/daloradius/login.php
default username and password of dolaRadius is:

Username: administrator
Password: radius

```
