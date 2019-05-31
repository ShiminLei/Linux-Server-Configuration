# Introduction

This is a guideline about how Linux Server Configuration.

Access server by :

`ssh -i .ssh/linuxCourse grader@54.236.241.124 -p 2200`

The website can be visited on :

[http://54.236.241.124/](http://54.236.241.124/)

The project can be seen in :

https://github.com/ShiminLei/Item-Catalog

# Tasks

#### Virtual Linux

```bash
vagrant init ubuntu/trusty64
vagrant up
vagrant ssh
```

#### Get your server

1. Start a new Ubuntu Linux server instance on [Amazon Lightsail](https://lightsail.aws.amazon.com/). 

2. SSH into your server.

   ```bash
   ssh -i .ssh/LightsailDefaultKey-us-east-1.pem ubuntu@54.236.241.124
   ```

#### Secure your server

1. Update all currently installed packages

   ```bash
   sudo apt-get update  # check if there any can be updated
   sudo apt-get upgrade # update 
   sudo apt-get autoremove  # remove redundant package
   
   # if the number of updated package if wrong, use
   sudo aptitude update
   sudo aptitude safe-upgrade
   ```

2. Change the SSH port from **22** to **2200**. Make sure to configure the Lightsail firewall to allow it.

   **Note**: You need to allow all port on Amazon Lightsail first - [question link](https://knowledge.udacity.com/questions/18052)

   ```bash
   sudo nano /etc/ssh/sshd_config  # then change Port 22 to Port 2200 (uncommit#), save and quit
   sudo service ssh restart  # reload SSH ,使修改生效
   # now can ssh with
   ssh -i .ssh/LightsailDefaultKey-us-east-1.pem ubuntu@54.236.241.124 -p 2200
   ```

3. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

   ```bash
   sudo ufw status
   sudo ufw default deny incoming  # 设置默认值 拒绝传入
   sudo ufw default allow outgoing # 设置默认值 允许传出连接
   
   # sudo ufw allow ssh 
   sudo ufw allow 2200/tcp
   sudo ufw allow 80/tcp
   sudo ufw allow 123/udp
   sudo ufw enable 
   
   # 关闭是 deny  
   # 删除是 delete allow
   ```

#### Give `grader` access

1. Create a new user account named `grader`

   ```bash
   sudo adduser grader  # password: udacity
   ```

2.  Give `grader` the permission to `sudo`

   ```bash
   sudo ls /etc/sudoers.d # check sudoer
   sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
   sudo nano /etc/sudoers.d/grader   # change ubuntu to grader
   
   # check if successful
   su grader
   sudo cat /etc/passwd 
   # 如果grader不是sudoer, 那么会报错：grader is not in the sudoers file.  This incident will be reported
   exit # 退出 grader, 回到 ubuntu
   ```

3. Create an SSH key pair for `grader` using the `ssh-keygen` tool

   ```bash
   # in locel /Users/leishimin/.ssh
   ssh-keygen
   # Your identification has been saved in linuxCourse.
   # Your public key has been saved in linuxCourse.pub.
   cat .ssh/linuxCourse.pub
      
    # in grader 
   mkdir .ssh
   touch .ssh/authorized_keys  # 建立空文件
   nano .ssh/authorized_keys # copy things in .ssh/linuxCourse.pub
   # 加一些权限之类的保密措施
   chmod 700 .ssh
   chmod 644 .ssh/authorized_keys
   
   # now you can login grader with below:
   ssh -i .ssh/linuxCourse grader@54.236.241.124 -p 2200
   
   # it is more secure to set ProhibitRootLogin to NO and be uncommented because a password can be brute forced
   cat /etc/ssh/sshd_config | egrep "PermitRootLogin"
   # then uncommit #PermitRootLogin prohibit-password
   ```
   
### Prepare to deploy your project.

1. Configure the local timezone to UTC.

   ```bash
   sudo dpkg-reconfigure tzdata
   # Then chose 'None of the above', then UTC.
   ```

2. Install and configure Apache to serve a Python mod_wsgi application.

   **Note**: If you built your project with Python 3, you will need to install the Python 3 mod_wsgi package on your server: `sudo apt-get install libapache2-mod-wsgi-py3`.

   ```bash
   sudo apt-get install apache2
   sudo apt-get install python-setuptools libapache2-mod-wsgi
   sudo apt-get install libapache2-mod-wsgi-py3
   sudo service apache2 restart
   ```

3. Install and configure PostgreSQL:

   - Do not allow remote connections

   - Create a new database user named `catalog` that has limited permissions to your catalog application database.

     ```bash
     sudo apt-get install postgresql
     # Check if no remote connections are allowed 
     sudo vim /etc/postgresql/9.3/main/pg_hba.conf
     
     sudo su - postgres
     # create database user
     psql
     postgres=# CREATE DATABASE catalog;
     postgres=# CREATE USER catalog;
     # Set password for user catalog
     postgres=# ALTER ROLE catalog WITH PASSWORD 'password';
     # Give user "catalog" permission to "catalog" application database
     postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
     # Quit postgreSQL 
     postgres=# \q
     # Exit from user "postgres"
     exit
     
     # 下次用的时候 用catalog登陆  密码 password
     psql -U catalog -d catalog -h 127.0.0.1 -p 5432
     ```

4. Install `git`.

   ```bash
   sudo apt-get install git
   ```

   ### Deploy the Item Catalog project.

1. Clone and setup your **Item Catalog** project from the Github repository you created earlier in this Nanodegree program.

   ```bash
   cd /var/www
   sudo mkdir FlaskApp
   cd FlaskApp
   sudo git clone https://github.com/ShiminLei/Item-Catalog.git
   cd Item-Catalog
   # Rename project.py to __init__.py using sudo mv project.py __init__.py    如果在本地改过了，这里就不用改了
   # then change db in all .py file to 
   # engine = create_engine('postgresql://catalog:password@localhost/catalog')
   
   # disable .git
   cd /var/www/FlaskApp
   sudo nano .htaccess
   # paste
   RedirectMatch 404 /\.git
   
   sudo apt-get install apache2
   sudo apt-get install python-pip
   sudo pip install -r requirements.txt
   sudo apt-get -qqy install postgresql python-psycopg2
   sudo apt-get install libapache2-mod-wsgi
   # sudo apt-get install libapache2-mod-wsgi-py3
   
   
   # create database
   sudo python database_setup.py
   sudo python lotsofitems.py
   ```

2. Set it up in your server so that it functions correctly when visiting your server’s IP address in a browser. Make sure that your `.git` directory is not publicly accessible via a browser!

   ```bash
   sudo nano /etc/apache2/sites-available/FlaskApp.conf
   # paste below in conf
   <VirtualHost *:80>
   	ServerName 54.236.241.124
   	ServerAdmin sl2kd@virginia.edu
   	WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
   	<Directory /var/www/FlaskApp/ItemCatalog/>
   		Order allow,deny
   		Allow from all
   	</Directory>
   	Alias /static /var/www/FlaskApp/ItemCatalog/static
   	<Directory /var/www/FlaskApp/ItemCatalog/static/>
   		Order allow,deny
   		Allow from all
   	</Directory>
   	ErrorLog ${APACHE_LOG_DIR}/error.log
   	LogLevel warn
   	CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
   
   
   # Enable the virtual host with the following command
   sudo a2ensite FlaskApp
   # sudo a2dissite FlaskApp
   # sudo a2dissite 000-default
   sudo systemctl reload apache2
   
   # Create the .wsgi file 
   cd /var/www/FlaskApp
   sudo nano flaskapp.wsgi
   # then paste
   #!/usr/bin/python
   import sys
   import logging
   logging.basicConfig(stream=sys.stderr)
   sys.path.insert(0,"/var/www/FlaskApp/")
   from ItemCatalog import app as application
   application.secret_key = 'Add your secret key'
   
   
   # restart apache
   sudo service apache2 restart
   
   # Note: 查看报错可以看
   sudo cat /var/log/apache2/error.log
   
   ```

   

# Reference

How to install pip3

https://www.jianshu.com/p/2cfdfd246018   

How to deploy Flask on Linux

https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps