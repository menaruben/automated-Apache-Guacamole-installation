# Documentation Guacamole Server
I have updated, automated and followed the instructions of this [documentation](https://www.byteprotips.com/post/apache-guacamole-1-1-0-install-guide)
## Default password guacamole
user: guacadmin
pass: guacadmin

vm: root, guacadmin
## History of commands:
```:
# install dependencies & set hostname:
hostnamectl set-hostname guac
sudo su -
yum install -y epel-release

yum -y localinstall --nogpgcheck https://download1.rpmfusion.org/free/el/rpmfusion-free-release-7.noarch.rpm https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-7.noarch.rp

yum install -y cairo-devel libjpeg-turbo-devel libwebsockets-devel libpng-devel uuid-devel ffmpeg-devel freerdp-devel pango-devel libssh2-devel libvncserver-devel pulseaudio-libs-devel openssl-devel libvorbis-devel libwebp-devel libtool libtelnet-devel freerdp mariadb-server wget tomcat

yum -y update
yum -y upgrade

# Download and extract Guacamole server source code (.tar.gz) and download Guacamole web app (.war):
wget https://downloads.apache.org/guacamole/1.1.0/source/guacamole-server-1.1.0.tar.gz
tar -xzf guacamole-server-1.1.0.tar.gz
wget https://downloads.apache.org/guacamole/1.1.0/binary/guacamole-1.1.0.war

# prepare for compiling and installation
cd guacamole-server-1.1.0
./configure --with-init-dir=/etc/init.d

# compile and install guacamole
make install
ldconfig && cd ~

systemctl enable tomcat && systemctl enable mariadb && systemctl enable guacd

cp ~/guacamole-1.1.0.war /var/lib/tomcat/webapps/guacamole.war

    # only if firewalld is running:
# firewall-cmd --permanent --add-port=8080/tcp
# firewall-cmd --permanent --add-port=4822/tcp
# firewall-cmd --reload

mkdir -p /usr/share/tomcat/.guacamole/{extensions,lib}
wget https://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-8.0.28.tar.gz
tar -xzf mysql-connector-java-8.0.28.tar.gz
cp mysql-connector-java-8.0.28/mysql-connector-java-8.0.28.jar /usr/share/tomcat/.guacamole/lib/
wget https://downloads.apache.org/guacamole/1.1.0/binary/guacamole-auth-jdbc-1.1.0.tar.gz
tar -xzf guacamole-auth-jdbc-1.1.0.tar.gz
cp guacamole-auth-jdbc-1.1.0/mysql/guacamole-auth-jdbc-mysql-1.1.0.jar /usr/share/tomcat/.guacamole/extensions/

systemctl start mariadb  && systemctl start tomcat
mysql_secure_installation #enter because no password set -> new password y
#       new password = guacdemo
# remove anonymous users?: y
# disallow root login remotely?: y
# remove test database and access to it?: y
# reload privileges tables: y

# login to mysql and make database
mysql -u root -p # enter "guacdemo"
# MariaDB [(none)]>:
CREATE DATABASE IF NOT EXISTS guacdb DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
GRANT SELECT,INSERT,UPDATE,DELETE ON guacdb.* TO 'guacuser'@'localhost' IDENTIFIED BY 'guacpass' WITH GRANT OPTION;
flush privileges;
quit

# install more things..
wget https://downloads.apache.org/guacamole/1.1.0/source/guacamole-client-1.1.0.tar.gz
tar -xzf guacamole-client-1.1.0.tar.gz
cat guacamole-client-1.1.0/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-mysql/schema/*.sql | mysql -u root -p guacdb # password = guacdemo

mkdir -p /etc/guacamole/ && vi /etc/guacamole/guacamole.properties
# guacamole.properties content:
# MySQL properties
mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacdb
mysql-username: guacuser
mysql-password: guacpass
#Additional settings
mysql-default-max-connections-per-user: 0
mysql-default-max-group-connections-per-user: 0
# end

chmod 0400 /etc/guacamole/guacamole.properties
chown tomcat:tomcat /etc/guacamole/guacamole.properties
ln -s /etc/guacamole/guacamole.properties /usr/share/tomcat/.guacamole/
chown tomcat:tomcat /var/lib/tomcat/webapps/guacamole.war

vi /etc/my.cnf # -> add line under socket=..
default-time-zone='+2:00'

setsebool -P tomcat_can_network_connect_db on
restorecon -R -v /usr/share/tomcat/.guacamole/lib/mysql-connector-java-8.0.28.jar

# after reboot go to your browser and enter [ip]:8080/guacamole
reboot

```

## Clone VM
C:\Program Files\Oracle\VirtualBox>VBoxManage clonemedium disk --format VMDK --variant Standard "C:\Users\50MER211\VirtualBox VMs\GUACAMOLE_centos7\GUACAMOLE_centos7.vdi" "C:\Users\50MER211\VirtualBox VMs\GUACAMOLE_centos7\GUACAMOLE_centos7.vmdk"
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Clone medium created in format 'VMDK'. UUID: 551e52e8-503c-4fdf-90b4-944c1ffe19cc

C:\Program Files\Oracle\VirtualBox>
## Change to DHCP on centos7
nmtui
