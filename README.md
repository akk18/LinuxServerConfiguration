# LinuxServerConfiguration
UFullStack-Project6

## 1 Login instructions for user : `grader`
Authenticate to the server : 35.161.192.163 using private SSH key shared in the "Notes to Reviewer" field. Note that Authentication using password has been disabled.
Steps:
-Create a file (say grader) : vi grader
-Copy the private SSH key shared into this ‘grader’ file.
-Run the command to create a ssh session with the server:  `$ ssh -i <pathToGrader-File> grader@35.161.192.163  -p 2200 `

## 2 Web Application Details:

Catalog  App IP:  <a href="http://35.161.192.163/">http://35.161.192.163/</a>
SSH Port : 2200
Please follow these steps to properly submit this project:
	1.	Create a new GitHub repository and add a file named README.md. 
	2.	Your README.md file should include all of the following: i. The IP address and SSH port so your server can be accessed by the reviewer. ii. The complete URL to your hosted web application. iii. A summary of software you installed and configuration changes made. iv. A list of any third-party resources you made use of to complete this project. 
	3.	Locate the SSH key you created for the grader user. 
	4.	During the submission process, paste the contents of the grader user's SSH key into the "Notes to Reviewer" field. 


## 3 Software installed

Update all currently installed packages : `$ sudo apt-get update`
Upgrade all installed packages: `$ sudo apt-get upgrade`

Install software:

```
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi
$ sudo apt-get install postgresql
$ sudo apt-get install git

$ sudo apt-get install python python-pip
$ sudo apt-get install libapache2-mod-wsgi python-dev
$ pip install flask
 Notice: Do not use sudo with pip.
$ sudo pip2 install packaging oauth2client redis passlib flask-httpauth
$ sudo pip2 install sqlalchemy flask-sqlalchemy psycopg2 bleach requests

```

## 4 configuration changes

### 4.1 Configure ssh to restrict root login and login using password.

Open `/etc/ssh/sshd_config` and modify:
```
PermitRootLogin no
PasswordAuthentication no
```

Restart ssh service:
`$ sudo service ssh restart`

###4.2  Add new user `grader` and give `sudo` permission

Connect as `ubuntu` using your AWS accounts default private key - LightsailDefaultPrivateKey-us-west-2.pem:
`$ ssh -i LightsailDefaultPrivateKey-us-west-2.pem ubuntu@35.161.192.163 `	

Add user `grader`:
`$ sudo adduser grader sudo`

### 4.3 Configure key based authentication for `grader`

Generate key pairs on local machine:
`$ ssh-keygen`

Get two key files `grader`, `grader.pub` generated by above command and copy grader.pub to the server with below command lines:

```
$ mkdir .ssh
$ touch .ssh/authorized_keys
```

Copy the content of `grader.pub` to `authorized_keys`
Setup key file permissions:

```
$ chmod 700 .ssh
$ chmod 644 .ssh/authorized_keys
$ sudo chown -R grader:grader /home/grader/.ssh
```

###4.4 Configure the local timezone to UTC
`sudo timedatectl set-timezone Etc/UTC`


### 4.4 Configure PostgreSQL

Change current user to `postgres`:
`$ sudo su - postgres`

Login PostgreSQL console:
`$ psql`

Set password for `postgres`:
`\password postgres`

Create database user `catalog` and set password:
`CREATE USER catalog WITH PASSWORD 'password';`

Create database `moviedb` and set its owner to be `catalog`:
`CREATE DATABASE moviedb OWNER catalog;`

Grant privilege on `moviedb` to `catalog`:
`GRANT ALL PRIVILEGES ON DATABASE moviedb to catalog;`

Do not allow remote connections:
Check configuration file `/etc/postgresql/9.5/main/pg_hba.conf`

It only contains the following four lines

```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

### 4.5 Deploy Catalog App

Clone project repo from github to `/var/www/`:

`$ git clone https://github.com/akk18/itemCatalog`

Modifications for this project:

in `database_setup.py` :

```
-engine = create_engine('sqlite:///movies.db')
+engine = create_engine('postgresql://catalog:password@localhost/catalogdb')
```

in `application.py`:

```
-CLIENT_ID = json.loads(open('client_secrets.json', 'r').read())['web']['client_id']
+CLIENT_ID = json.loads(open('/var/www/catalog/client_secrets.json', 'r').read())['web']['client_id']
-        oauth_flow = flow_from_clientsecrets('client_secrets.json', scope='')
+        oauth_flow = flow_from_clientsecrets('/var/www/catalog/client_secrets.json', scope='')
```

Also move `app.secret_key` out of `if __name__ == '__main__':`

```
-    app.secret_key = 'super_secret_key'
     app.debug = True
-    app.run(host='0.0.0.0', port=8000)
+    app.run()
```

Login <a href="https://console.developers.google.com/apis/credentials">Google Developers Console</a>, add `http://35.161.192.163` to `Authorized JavaScript origins`, and update `client_secrets.json`.

Create a new file `catalog.wsgi` in `/var/www/catalog/`:

```
#!/usr/bin/python
import sys
sys.path.insert(0,"/var/www/catalog/")

from application import app as application
application.secret_key = 'super_secret_key'
``` 

Modify `000-default.conf` in `/etc/apache2/sites-enabled/`:

```
<VirtualHost *:80>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	#ServerName www.example.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/catalog
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi

 	<directory /var/www/catalog>
		WSGIApplicationGroup %{GLOBAL}
		Order deny,allow
		Allow from all
	</directory>
	# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
	# error, crit, alert, emerg.
	# It is also possible to configure the loglevel for particular
	# modules, e.g.
	#LogLevel info ssl:warn

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

	# For most configuration files from conf-available/, which are
	# enabled or disabled at a global level, it is possible to
	# include a line for only one particular virtual host. For example the
	# following line enables the CGI configuration for this host only
	# after it has been globally disabled with "a2disconf".
	#Include conf-available/serve-cgi-bin.conf
</VirtualHost>
``` 

Restart Apache Server:
`$ sudo apache2ctl restart`

### 4.6 Configure Firewall

Open `/etc/ssh/sshd_config` and modify line `Port 22` to `Port 2200`

Run following command lines to configure UFW:

```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow www
$ sudo ufw allow ntp
$ sudo ufw enable
```

## 5 Third-party resources
<a href="https://askubuntu.com/questions/16650/create-a-new-ssh-user-on-ubuntu-server">Create a new SSH user on Ubuntu Server</a>
<a href="https://unix.stackexchange.com/a/179956">Add user to sudo group</a>
<a href="http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/">Flask Documentation - mod_wsgi (Apache)</a>
<a href="https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps">How To Secure PostgreSQL on an Ubuntu VPS</a>
<a href="https://www.thegeekstuff.com/2010/09/change-timezone-in-linux/">How To: 2 Methods To Change TimeZone in Linux</a>
