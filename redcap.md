# Workbook for Redcap
```
dmesg, sudo -i, passwd,
```
## install of Redcap
Redcap is 


### Install of required package
Combine to the host: for linux, creat the 'config' file under ~/.ssh, where the privat key was saved, and give the configuration information there. Then in terminal connect with ssh.

/server/shell:
```
apt update

apt install apache2 php php-mysql libapache2-mod-php mysql-server composer curl php-curl php-xml php-zip python3-pip php-gd imagemagick php-imagick sendmail postfix

pip3 install glances

mysql_secure_installation
```
### Transfer Redcap
In shell of control computer:

```
scp C\:Download\RedcapFile.zip username@server:/ordner
```

!Attention: the path: server/ordner must have permission to write. To do that, you can use this command in shell of server:
```
sudo chmod 755 server/ordner

# after changed the permission, redo the transfer part, and the do this step:

cd /var/www/html
sudo unzip server/ordner/RedcapFile.zip
```
### Creat and configuration sql database for redcap
!Path of mysql configuration file: `/etc/mysql/my.cnf`

Then in shell of server:
```
# backup the sql configuration file
sudo cp /etc/mysql/mycnf/my.cnf /etc/mysql/my.cnf.backup
# edit configuration file, add the followed line:

[mysqld]
# if you want to change the sql data save path.
datadir=/mnt/redcap-hdd/mysqldata

# other configuration
max_allowed_packet=1G
innodb_buffer_pool_size=2G
read_rnd_buffer_size=4M
sort_buffer_size=4M

```
Then creat the new sql account and database, you need to get into mysql from the control computer shell as root user:
```
sudo mysql 
```

Then in mysql commandline:
```
CREATE DATABASE redcap;
CREATE USER 'redcapuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON redcap.* TO 'redcapuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
#### Extra Option 1: change the path where to store the sql data.
the easiest way to do that is to creat a symbolink(softlink) between the default path and your choosed path. You can do that with this command line:
```
sudo systemctl stop mysql # stop the mysql server
sudo mv /var/lib/mysql /new/path/ # move the data directory to new path you choosed
cd /var/lib/mysql # test if the old directory still exist, it shouldn't
sudo ln -s /new/path/mysql /var/lib/mysql # creat the softlink
sudo chown -R mysql:mysql /var/lib/mysql # change the owner of the softlink directory, its owner should also be mysql.
```
also, you in Ubuntu there is a process called Apparmor, it control which app can access which file or directory. So we must add the directory into the safelist of Apparmor:
```
vim /etc/apparmor.d/usr.sbin.mysqld # access the safelist of Apparmor for mysqld
``` 
and then add the followed code:
```
/new/path/mysql/ r,
/new/path/mysql/** rwk,
```
then save and exit, and reload the apparmor:
```
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.mysqld
```
#### Extra Option 2: what if you changed the redcap_base_url at setting process and can't open the redcap from explore anymore?
The Setting of redcap_base_url was saved in the table `redcap_config` as the `field_name='redcap_base_url'`. Just get into mysql, and the redcap database, then update the value with: `UPDATE redcap_config SET VALUE = 'ip-related url' WHERE field_name='redcap_base_url'`
#### useful code for mysql
1. get into mysql: ```mysql -u [username] -p```. [username] must be replaced by your username, and password will be promopt. To sign in as root: sudo mysql.
2. list the database: ```SHOW DATABASES;```
3. select database: ```USE [database_name];```
4. show table： ```SHOW TABLES;```
5. exit current table or database: ```EXIT;``` or ```QUIT;```
6. check the password policy: ```SHOW VARIABLES LIKE 'validate_password%';```, and then ```SET GLOBAL validate_password.policy=LOW```
7. get into sql and skip the root: ```sudo mysqld_safe --skip-grant-table&``` and then ```sudo mysql --user=root mysql```
8. remove mysql:
```
sudo systemctl stop mysql
sudo apt purge mysql-server mysql-client mysql-common mysql-server-core-* mysql-client-core-*
sudo rm -rf /etc/mysql /var/lib/mysql
sudo apt autoremove
sudo apt autoclean
```
9. show describtion of certain table: `DESCRIBE table_name;`, the output should be like this:
```
+------------+--------------+------+-----+---------+-------+
| Field      | Type         | Null | Key | Default | Extra |
+------------+--------------+------+-----+---------+-------+
| field_name | varchar(191) | NO   | PRI |         |       |
| value      | mediumtext   | YES  |     | NULL    |       |
+------------+--------------+------+-----+---------+-------+
```
10. select the value: `SELECT value FROM table_name WHERE field_name='your serach field';`

### Configuration of the redcap database.php
goto `/var/www/html/redcap/database.php` and find the `$hostname $db $username $password` and change it. Then open internet explore, give in the URL: `ipaddress/redcap/install.php` and follow the guidline.

### Configuration of cron job.
open the configuration file by tipp this: `sudo crontab -u www-data -e`, and then choose the editor you like to edit. Add this line with editor: `* * * * * /usr/bin/php /var/www/html/redcap/cron.php > /dev/null 2>&1`. Here, every one * is one minitues. And then restart your webserver.

### Permission Staff
```
sudo chown -R www-data:www-data /var/www/html/redcap/temp/
sudo chmod -R 755 /var/www/html/redcap/temp/
sudo chown -R www-data:www-data /var/www/html/redcap/modules/
sudo chmod -R 755 /var/www/html/redcap/modules/

# if you changed the upload file storage place, change the path followed to that path.
sudo chown -R www-data:www-data /var/www/html/redcap/edocs/
sudo chmod -R 755 /var/www/html/redcap/edocs/
```

### Configuration of ImageMagic
open the policy file under `/etc/ImageMagick-6/policy.xml`, and change the PDF mode from none to read. The line looks like this:`<policy domain="coder" rights="read" pattern="PDF" />`

### Configuration of PHP
find the php.ini: `/etc/php/8.1/apache2/php.ini`
setup the fellowing variable:
```
max_input_vars=100000
upload_max_filesize=128M
post_max_size=32M

sendmail_path = /usr/sbin/sendmail -t -i # for the next step, config the sendmail

# after of the setting of SSL: 
session.cookie_secure = on
```
### configuration of mail service
you should also install the postfix: `sudo apt install postfix`.
All the Email which sended out was saved under the table `redcap_outgoing_email_sms_log` in mysql, in case you need the password enmergency.

### Configuration of SSL
在这里，我假设你已经会配置基本的/etc/apache2/sites-available/000-default.conf这个文件来达到已经可以通过 http 的方式来访问你的站点。

在/etc/apache2这个目录下，有两个有关的目录sites-available和sites-enabled，我们进入sites-enabled目录下可以发现，里面有一个文件000-default.conf
```
$ ll 
lrwxrwxrwx 1 root root 35 Dec 28 15:24 000-default.conf -> ../sites-available/000-default.conf
```
实质上这个文件是/etc/apache2/sites-available/000-default.conf这个文件的软链接。

我们要配置另 ssl 证书，要依靠另一个文件，也就是default-ssl.conf，首先我们需要设置一个软链接，把这个文件链接到sites-enabled这个文件夹中:

```
ln -s /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-enabled/000-default-ssl.conf
```

然后去修改这个文件000-default-ssl.conf，因为已经做了软链接，其实这时候修改000-default-ssl.conf或default-ssl.conf都一样。

这个文件没有做任何修改前长这样子（去除自带的注释之后）：
```
<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
        ServerAdmin webmaster@localhost

        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        SSLEngine on

        SSLCertificateFile    /etc/ssl/certs/ssl-cert-snakeoil.pem
        SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
    
        <FilesMatch "\.(cgi|shtml|phtml|php)$">
                SSLOptions +StdEnvVars
        </FilesMatch>
        <Directory /usr/lib/cgi-bin>
                SSLOptions +StdEnvVars
        </Directory>

    </VirtualHost>
</IfModule>
```
然后把从阿里云上面下载好的证书(3个文件)传到你自定义的目录中

然后我们需要修改一下，修改成这样:
```
<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
        ServerAdmin 你的邮箱
        
        DocumentRoot /var/www/你的目录
        ServerName 你的域名

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        SSLEngine on
        # 注意，需要添加这三行
        SSLCertificateFile 你自定义的路径/2_xxx.xxx.xxx.crt
        SSLCertificateKeyFile 你自定义的路径/3_xxx.xxx.xxx.key
        SSLCertificateChainFile 你自定义的路径/1_root_bundle.crt
    
        <FilesMatch "\.(cgi|shtml|phtml|php)$">
                SSLOptions +StdEnvVars
        </FilesMatch>
        <Directory /usr/lib/cgi-bin>
                SSLOptions +StdEnvVars
        </Directory>
    </VirtualHost>
</IfModule>
```
改好之后保存。

然后这时，我们加载一下 Apache2 的 SSL 模块：

```
sudo a2enmod ssl   #加载模块
sudo service apache2 restart # 重启服务
```

这时，在浏览器输入https://你的域名应该已经可以通过 https 的方式来访问网站了，这时浏览器那里应该也已经有了一个绿色的小锁。

但是，但是…这还不够，因为我们如果不主动输入https://的话，直接输入域名，还是会直接跳转到 80 端口的普通的 http 方式访问，所以我们需要强制使用 https 来访问

强制使用https
我们只需要打开/etc/apache2/sites-available/000-default.conf这个文件，在你的这个标签内随便一个地方加上三行：
```
RewriteEngine on
RewriteCond   %{HTTPS} !=on
RewriteRule   ^(.*)  https://%{SERVER_NAME}$1 [L,R]

```
然后保存,然后启动 Apache2 的重定向, and restart the Apache2：
```
 sudo a2enmod rewrite
 sudo service apache2 restart
```
## external module
normally, the external module can be instanced very easily through the build-in External Module Manager. But for the REDCap in the clinical server it works not because of the firewall. In this case, the manuel insatance was be required. Here is what you can do.

1. Download the module from Github.
2. Unzip it into the /var/www/html/redcap/modules
3. Rename it into this format: name_of_module_v1.0.0
```
sudo chmod 777 /var/www/html/redcap/modules/name_of_module_v1.0.0 

sudo chown www-data:www-data /var/www/html/redcap/modules/name_of_module_v1.0.0
```
4. Change the permission of module directory. 
5. Enable the module in Module Manager inside Redcap.

An Example Structure:

```
redcap
|-- modules
|   |-- my_module_name_v1.0.0
|   |-- other_module_v2.9
|   |-- other_module_v2.10
|   |-- other_module_v2.11
|   |-- yet_another_module_v1.5.3
|-- redcap_vX.X.X
|-- redcap_connect.php
|-- ...
``` 

### Speicial Notice
There are some special configuration or changes should be done for some module. In this section this thing will be described.

#### Orca Search Module
something special should be noticed for this Module: 
1. if you don't have composer installed yet: ```sudo apt install composer```
2. if you don't have smarty installed yet: ```composer requir smarty/smarty```
3. in the file OrcaSearch.php, line 7 should be changed into ```../vendor/autoload.php```
4. Enable the module in Control Center
5. Configure the module in Certain project.

## Set your own Email server
1. Install postfix, and mailutils: `apt install postfix mailutils`
2. configure the postfix as installing, or reconfigure that use this command: `dpkg-reconfigure postfix`
3. Setting should be like this: 
```
mode: internet fiel
mailname: same as your domain, here is: dir-redcap.med.uni-heidelberg.de
destinations: dir-redcap.med.uni-heidelberg.de, localhost
mynetworks (who can permit this email server): 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
``` 
about the mynetworks: 
a) 127.0.0.0/8: This is for IPv4. It means that any IP address that starts with 127 is allowed to relay mail. In practical terms, this is usually just the loopback interface 127.0.0.1, aka localhost.

b) [::ffff:127.0.0.0]/104: This is an IPv6-mapped IPv4 address. This means that the IPv4 loopback can be accessed via this IPv6 address. Again, this is a form of localhost but in an IPv6 context.

c) [::1]/128: This is the loopback address for IPv6, equivalent to 127.0.0.1 in IPv4.

4. Test with: `echo "this is a email" | mail -s "test email" target@domain.com`
