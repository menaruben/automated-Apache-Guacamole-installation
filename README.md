#  automated-Apache-Guacamole-installation
This is an automated installation of apache guacamole 1.5.0 (on Centos). I have added a vagrant vm inside the [src](./src/) folder for testing purposes.

# Documentation
## annotation
Please check if the script contains the latest [guacamole](https://downloads.apache.org/guacamole/) and latest [mysql-connector](https://dev.mysql.com/downloads/connector/net/8.0.html) versions. You can compare the latest versions with these two variables of the [installation script](./src/guacamole.sh):
```bash:
guacversion="1.5.0"
connectorversion="8.0.32"
```

## apache guacamole default credentials
After running the [installation script](./src/guacamole.sh) you can open the guacamole server on your web browser with ```localhost:8080/guacamole```. If you are using the [vagrant vm](./src/Vagrantfile) then you'll need to open ```localhost:8081/guacamole```. The default credentials for guacamole are:
- user: guacadmin
- pass: guacadmin

You can change the username/password or add other users on the ```http://localhost:8080/guacamole/#/settings/users``` page. I would highly recommend adding another user and removing the guacadmin user for better security.

## installation script
```bash:
hname="guac"
SQLpasswd="guacpass"
guacversion="1.5.0"
connectorversion="8.0.32"

cd ~
# SET HOSTNAME
# read -p "Please enter the hostname you would want your machine to have: " hname
hostnamectl set-hostname $hname

# read -p "Please enter the sql password: " SQLpasswd

# GET DEPENDENCIES
yum install -y epel-release
yum update -y

# install rpm-fusion
yum -y localinstall --nogpgcheck https://download1.rpmfusion.org/free/el/rpmfusion-free-release-7.noarch.rpm https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-7.noarch.rpm

# install dependencies
yum install -y cairo-devel libjpeg-turbo-devel libwebsockets-devel libpng-devel uuid-devel ffmpeg-devel freerdp-devel pango-devel libssh2-devel libvncserver-devel pulseaudio-libs-devel openssl-devel libvorbis-devel libwebp-devel libtool libtelnet-devel freerdp mariadb-server wget tomcat

# download and extract the Guacamole server source code (.tar.gz) and download the Guacamole Web Application (.war)
wget https://downloads.apache.org/guacamole/$guacversion/source/guacamole-server-$guacversion.tar.gz
tar -xzf guacamole-server-$guacversion.tar.gz
wget https://downloads.apache.org/guacamole/$guacversion/binary/guacamole-$guacversion.war

# compile and install
cd guacamole-server-$guacversion
./configure --with-init-dir=/etc/init.d
make install
ldconfig && cd ~

# enable tomcat, mariadb, and guacd 
systemctl enable tomcat && systemctl enable mariadb && systemctl enable guacd

# copy guacamole web app to correct directory
cp ~/guacamole-$guacversion.war /var/lib/tomcat/webapps/guacamole.war

# open port 8080 for tomcat/guacamole
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

# download and configure MySQL
mkdir -p /usr/share/tomcat/.guacamole/{extensions,lib}
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-$connectorversion.tar.gz
tar -xzf mysql-connector-j-$connectorversion.tar.gz
cp mysql-connector-j-$connectorversion/mysql-connector-j-$connectorversion.jar /usr/share/tomcat/.guacamole/lib/
wget https://downloads.apache.org/guacamole/$guacversion/binary/guacamole-auth-jdbc-$guacversion.tar.gz
tar -xzf guacamole-auth-jdbc-$guacversion.tar.gz
cp guacamole-auth-jdbc-$guacversion/mysql/guacamole-auth-jdbc-mysql-$guacversion.jar /usr/share/tomcat/.guacamole/extensions/

# start mariadb and tomcatw
systemctl start mariadb  && systemctl start tomcat

# mysql_secure_installation
mysql -e "UPDATE mysql.user SET Password = PASSWORD('$SQLpasswd') WHERE User = 'root'"
mysql -e "DROP USER ''@'localhost'"
mysql -e "DROP USER ''@'$(hostname)'"
mysql -e "DROP DATABASE test"
mysql -e "FLUSH PRIVILEGES"

# create database
mysql -u root -p$SQLpasswd -e "CREATE DATABASE IF NOT EXISTS guacdb DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci"
mysql -u root -p$SQLpasswd -e "GRANT SELECT,INSERT,UPDATE,DELETE ON guacdb.* TO 'guacuser'@'localhost' IDENTIFIED BY 'guacpass' WITH GRANT OPTION"
mysql -u root -p$SQLpasswd -e "FLUSH PRIVILEGES"

# download and extract the guacamole client and cat .sql files to mysql from inside the jbdc folder
wget https://downloads.apache.org/guacamole/$guacversion/source/guacamole-client-$guacversion.tar.gz
tar -xzf guacamole-client-$guacversion.tar.gz
cat guacamole-client-$guacversion/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-mysql/schema/*.sql | mysql -u root -p$SQLpasswd guacdb

# create guacamole configuration file
mkdir -p /etc/guacamole/
cat > /etc/guacamole/guacamole.properties << EOF
# MySQL properties
mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacdb
mysql-username: guacuser
mysql-password: guacpass
#Additional settings
mysql-default-max-connections-per-user: 0
mysql-default-max-group-connections-per-user: 0
EOF

# fix some file permissions and create a symbolic link
chmod 0400 /etc/guacamole/guacamole.properties
chown tomcat:tomcat /etc/guacamole/guacamole.properties
ln -s /etc/guacamole/guacamole.properties /usr/share/tomcat/.guacamole/
chown tomcat:tomcat /var/lib/tomcat/webapps/guacamole.war

# sepcify timezone in /etc/my.cnf
cp /etc/my.cnf /etc/my.cnf.bak
sudo sed -i "s/^default-time-zone.*/default-time-zone = '+1:00'/" /etc/my.cnf

# fix permission issue with SELinux so that it doesnt prevent guacamole from working
setsebool -P tomcat_can_network_connect_db on
restorecon -R -v /usr/share/tomcat/.guacamole/lib/mysql-connector-j-$connectorversion.jar

# rebooting message
echo "you can open your guacamole (localhost:8080/guacamole) after rebooting! :)"
```
