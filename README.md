# fullstack-nanodegree-webserver

You will take a baseline installation of a Linux distribution on a virtual machine and prepare it to host your web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

## For reviewer

ssh -p 2200 -i ~/.ssh/udacity_key.rsa grader@52.37.55.67

http://ec2-52-37-55-67.us-west-2.compute.amazonaws.com/

## Get started

The title enumeration reflects the steps outlined in the Udacity project instructions. Some steps between those major steps are necessary and described under titles without numbers.

### 1./2. Launch VM via Udacity Account and SSH into VM

Go to your Udacity account to launch your VM. Follow the link in Udacity project instructions. This will provide you with an IP address for your publicly accessible server at AWS. In my case 52.37.55.67.

```
me@local:~$ ssh -i ~/.ssh/udacity_key.rsa root@52.37.55.67
```

### 3. Create new user named grader

```
root@vm:~$ adduser grader 
``` 

Generate strong password automatically at http://passwordsgenerator.net/. Use default settings.

### 4. Give grader permission to sudo

```
root@vm:~$ vi /etc/sudoers.d/grader
```

Require grader to use its own password to sudo. Add these lines to file grader:

`/etc/sudoers.d/grader:`
```
grader ALL=(ALL) ALL
```

And save the file. Then change permissions to 440:

```
root@vm:~$ chmod 440 /etc/sudoers.d/grader
root@vm:~$ ll
total 16
drwxr-xr-x  2 root root 4096 Feb 28 19:26 ./
drwxr-xr-x 89 root root 4096 Feb 28 19:19 ../
-r--r-----  1 root root   30 Feb 28 19:26 grader
-r--r-----  1 root root  958 Feb 10  2014 README
```

#### Enable ssh key access for user grader

On the VM, copy root public ssh key into `~grader/.ssh/authorized_keys`. Change permissions
to `700 .ssh` and `644 authorized_keys`.
Now you can ssh into the VM as user grader:

```
me@local:~$ ssh grader@52.37.55.67 -i ~/.ssh/udacity_key.rsa
```

From now on only use user grader on the VM because it is saver. If root permissions are needed use sudo. 
If you get the warning message `sudo: unable to resolve host ip-10-20-42-11`. Then edit file `/etc/hosts` and add line `127.0.1.1 ip-10-20-42-11`
(replace ip-10-20-42-11 with your VM name).

### 5. Update all currently installed packages

```
grader@vm:~$ sudo apt-get update
grader@vm:~$ sudo apt-get upgrade
```

Answer `Y` on `Do you want to continue?`. (The following packages will be upgraded).

Answer `OK` on `Keep the local version currently installed?`. (/boot/grub/menu.lst).

### 6. Change the ssh port from 22 to 2200

Edit file `/etc/ssh/sshd_config`. Change line `Port 22` to `Port 2200`. Then reload ssh server:

```
grader@vm:~$ sudo reload ssh
```

Don't exit the machine yet. First try if you can connect as grader via the new SSH port:

```
me@local:~@ ssh -p 2200 -i ~/.ssh/udacity_key.rsa grader@52.37.55.67
```

### 7. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

```
grader@vm:~$ sudo ufw status
grader@vm:~$ sudo ufw default deny incoming
grader@vm:~$ sudo ufw default allow outgoing
grader@vm:~$ sudo ufw allow 2200
grader@vm:~$ sudo ufw allow www
grader@vm:~$ sudo ufw allow ntp
grader@vm:~$ sudo ufw enable
```

Note: Don't use `sudo ufw allow ssh` for ssh. This will only affect port 22. But we have switched to port 2200 already. Use `sudo ufw allow 2200` instead as above.

Check status of UFW:

```
grader@vm:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
80/tcp                     ALLOW       Anywhere
123                        ALLOW       Anywhere
2200                       ALLOW       Anywhere
80/tcp (v6)                ALLOW       Anywhere (v6)
123 (v6)                   ALLOW       Anywhere (v6)
2200 (v6)                  ALLOW       Anywhere (v6)
```

#### Prevent Root Login

Now that ssh for grader is fully configured and the firewall is enabled it's save to disable
the root access via ssh. In file `/etc/ssh/sshd_config` change setting `PermitRootLogin`
from `without-password` to `no`.

```
grader@vm:~$ sudo reload ssh
```

### 8. Configure local timezone to UTC

```
grader@vm:~$ sudo apt-get install ntp
grader@vm:~$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 services.quadra 64.71.128.26     3 u   54   64    1   31.719  -267.11   0.000
 pyramid.latt.ne 204.123.2.72     2 u   53   64    1   37.071  -267.02   0.000
 pegasus.latt.ne 130.207.244.240  2 u   52   64    1   88.909  -270.81   0.000
 i5.smatwebdesig 173.162.192.156  2 u   51   64    1   76.178  -267.23   0.000
 golem.canonical 192.150.70.56    2 u   50   64    1  137.958  -265.74   0.000
grader@vm:~$ date
Sun Feb 28 22:52:10 UTC 2016
```

### 9. Install and configure Apache to serve a Python mod_wsgi application

#### Install Apache2 webserver

```
grader@vm:~$ sudo apt-get install apache2
```

This will install and start the Apache2 webserver. By default pages are served from `/var/www/html/index.html`.  
Translate your IP to the AWS URL, as per instructions, like so:
`http://ec2-52-37-55-67.us-west-2.compute.amazonaws.com/'.
Open the default Apache page in a browser by going to this URL.

#### Install the application handler

```
grader@vm:~$ sudo apt-get install libapache2-mod-wsgi
```
Test the application handler. Create file `/var/www/html/myapp.wsgi`:

**myapp.wsgi**
```
def application(environ, start_response):
  status = '200 OK'
  output = 'Hello World'

  response_headers = [('Content-type', 'text/plain'), ('Content-Length', str(len(output)))]
  start_response(status, response_headers)

  return [output]
```

You then need to configure Apache to handle requests using this WSGI module. You do this by editing the `/etc/apache2/sites-enabled/000-default.conf` file.

For now, add the following line at the end of the `<VirtualHost *:80>` block, right before the closing `</VirtualHost>` line: `WSGIScriptAlias / /var/www/html/myapp.wsgi`.
And restart Apache to use the handler:

```
grader@vm:~$ sudo apache2ctl restart
```

Open your browser at your AWS URL and see `Hello World` welcome.

#### Tag existing project 3

Before you clone the project and make changes it might make sense to tag it. It's easy to add a full tag in git,
e.g. `git tag -a v1.0 -m "Project 3 complete SQLite version"`. There's a separate command to push tags,
`git push --tags`. Issue these commands from your local machine in the existing project 3 workarea.

#### Clone fullstack nanodegree project 3

Then install git on the server and clone the project:

```
grader@vm:~$ sudo apt-get install git
grader@vm:/var/www$ sudo git clone https://github.com/cg94301/fullstack-nanodegree-catalog.git
grader@vm:/var/www$ sudo ln -s fullstack-nanodegree-catalog/vagrant/catalog
```

#### Install dependencies for project 3

Use the Python package manager PIP to install dependencies.

```
grader@vm:~$ sudo apt-get install python-pip
grader@vm:~$ sudo pip install Flask
grader@vm:~$ sudo pip install SQLAlchemy
grader@vm:~$ sudo pip install oauth2client
```

You can test completeness of project 3 dependencies:

```
grader@vm:/var/www/catalog$ sudo python application.py
```

#### Create WSGI handler for catalog

**/var/www/catalog/catalog.wsgi**
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")
from application import app as application
application.secret_key = 'catalog_secret_key'
```

**/etc/apache2/sites-available**
```
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        ServerName 52.37.55.67

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/catalog

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        LogLevel info

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi

        <Directory /var/www/catalog/>
            Order allow,deny
            Allow from all
            Options -Indexes
        </Directory>
        Alias /static /var/www/catalog/static
        <Directory /var/www/catalog/static>
            Order allow,deny
            Allow from all
            Options -Indexes
        </Directory>
</VirtualHost>
```
Notice use of `Options -Indexes` to disable directory listings in Apache.

Switch from the default handler to the new catalog handler and reload:

```
grader@vm:~$ sudo a2dissite 000-default
grader@vm:~$ sudo a2ensite catalog
grader@vm:~$ sudo service apache2 reload
```

#### Get Catalog to work with SQLite first

As a first step, let's get the original project 3 to work. This project used SQLite. Only then switch to the required PostgreSQL setup.

You need to use absolute paths to your SQLite DB, like so:

`engine = create_engine('sqlite:////var/www/catalog/redwines.db?check_same_thread=False')`

Notice that you need 4 slashes for the absolute path. Also notice that we have disabled the check
for same thread.

Use absolute path for client secrets too:

`CLIENT_ID = json.loads(open('/var/www/catalog/client_secrets.json','r').read())['web']['client_id']`

Finally you need to set proper UNIX permissions. The Apache user is called `www-data`. This user needs to have
read and write permissions to the DB and the directory the DB is contained in.

#### Add client_secrets.json

You need client_secrets.json for oauth authorization. Go to Google Developers Console. 
Select the OAuth project. Select API Manager. Select Credentials. 
There you will find the web client defined in the OAuth project. Add 

Authorized JavaScript origins:

http://ec2-52-37-55-67.us-west-2.compute.amazonaws.com/

Authorized redirect URLs:

http://ec2-52-37-55-67.us-west-2.compute.amazonaws.com/login

http://ec2-52-37-55-67.us-west-2.compute.amazonaws.com/gconnect

Download the newly defined client secrets JSON file. Rename to client_secrets.json and use scp to copy to VM:

```
me@local:~$ scp -P 2200 -i ~/.ssh/udacity_key.rsa client_secrets.json grader@52.37.55.67:
```

#### Debug

Good commands to know for working with Apache2 webserver are:

```
sudo service apache2 reload
sudo service apache2 restart
sudo service apache2 stop
sudo service apache2 start
```

If you encounter an error, check the Apache error log at `/var/log/apache2/error.log`.

#### Tag Apache2 SQLite version of project 3

At this point project 3 is up and running with SQLite. Tag this project and push it to github. We'll call this v2.0.

#### Installing PostgreSQL

```
grader@vm:~$ sudo apt-get install postgresql postgresql-contrib
```

This command will install PostgreSQL 9.3 and also start the DB. You can access the DB with `psql` utility.

```
grader@vm:~$ sudo -u postgres psql
postgres=# CREATE USER catalog WITH CREATEDB LOGIN PASSWORD;
postgres=# \du
                             List of roles
 Role name |                   Attributes                   | Member of 
-----------+------------------------------------------------+-----------
 catalog   | Create DB                                      | {}
 postgres  | Superuser, Create role, Create DB, Replication | {}

postgres=# CREATE DATABASE catalog;
postgres=# \q
grader@vm:~$ sudo -u postgres createdb -O catalog catalog
grader@vm:~$ sudo -u postgres psql -U postgres -d catalog -c "REVOKE ALL ON SCHEMA public FROM public;"
grader@vm:~$ sudo -u postgres psql -U postgres -d catalog -c "GRANT ALL ON SCHEMA public TO catalog;"
```

#### Do not allow remote connections

This is actually already the default when installing PostgreSQL. Look into the host based
authentication file to double check settings:

`/etc/postgresql/9.3/main/pg_hba.conf:`
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                peer
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```

Configure the DB to listen to local addresses only.

`/etc/postgresql/9.3/main/postgresql.conf:`

```
listen_addresses = 'localhost'
```

#### Update SQLAlchemy to connect to PostgreSQL

You need psycopg2 library for Python:

```
grader@vm:~$ sudo apt-get install python-psycopg2
```

Change create engine dialect in SQLAlchemy to connect to PostgreSQL in files
`application.py`, `dbinit.py` and `dbpopulate.py`:

```
engine = create_engine('postgresql://catalog:caTal0g@localhost:5432/catalog')
```

Initialize the DB:

```
grader@vm:~$ python application.py
grader@vm:~$ python dbpopulate.py
```

Verify that everything is fine by going to http://ec2-52-37-55-67.us-west-2.compute.amazonaws.com/. 
Login, add wine, edit wine and delete wine. Everything should be working fine.

![catalog app top](images/awscatalog.png)

#### Tag final PostgreSQL version of project 3

At this point project 3 is up and running with PostgreSQL. Tag this project and push it to github. We'll call this v3.0.

*And you're done!*


## Optional Setup

### System Monitoring

(Source: https://pypi.python.org/pypi/Glances)

Glances is a cross-platform curses-based system monitoring tool written in Python.

```
grader@vm:~$ sudo apt-get install gcc python-dev
grader@vm:~$ sudo pip install psutil
grader@vm:~$ sudo pip install glances
grader@vm:~$ glances
```

![glances system monitor](images/glances.png)


### Automatic Updates

(Source: https://wiki.debian.org/UnattendedUpgrades)

Run Unattended Upgrades to keep the computer current with the latest security (and other) updates automatically. Monitor log files in /var/log/dpkg.log, or the files in /var/log/unattended-upgrades/.


```
grader@vm:~$ sudo apt-get install unattended-upgrades
```

The default configuration file is at `/etc/apt/apt.conf.d/50unattended-upgrades`. Defaults are fine for now.

Active updates via apt configuration file `/etc/apt/apt.conf.d/20auto-upgrades`.

`20auto-upgrades:`
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

### Protecting SSH with Fail2Ban

(Source: https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)

Install fail2ban and secure the configuration file from package upgrade
modifications by creating a local copy that fail2ban can find.

```
grader@vm:~$ sudo apt-get install fail2ban
grader@vm:~$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

In this file, there are a few settings you may wish to adjust. The settings located under the [DEFAULT] section will be applied to all services enabled for fail2ban that are not overridden in the service's own section.

`ignoreip = 127.0.0.1/8`

You can adjust the source addresses that fail2ban ignores by adding a value to the ignoreip parameter. Currently, it is configured not ban any traffic coming from the local machine. You can add additional addresses to ignore by appending them to the end of the parameter, separated by a space.

`bantime = 600`

The bantime parameter sets length of time that a client will be banned when they have failed to authenticate correctly. This is measured in seconds. By default, this is set to 600 seconds, or 10 minutes.

`findtime = 600`

`maxretry = 3`

The next two parameters that you want to pay attention to are findtime and maxretry. These work together to establish the conditions under which a client is found to be an illegitimate user that should be banned.

The maxretry variable sets the number of tries a client has to authenticate within a window of time defined by findtime, before being banned. With the default settings, the fail2ban service will ban a client that unsuccessfully attempts to log in 3 times within a 10 minute window.


Finally, we get to the portion of the configuration file that deals with individual services. These are specified by the section headers, like [SSH].

Each of these sections can be enabled by modifying or adding the enabled line to be "true":

`enabled = true`

By default, the SSH service is enabled and all others are disabled.

These sections work by using the default values we defined above. If you want to override any values, you can do so under the service's section. If you want to use the defaults, you aren't required to add anything.

To implement your configuration changes, you'll need to restart the fail2ban service. You can do that by typing:

```
grader@vm:~$ sudo service fail2ban restart
```

You can see all of your enabled jails by using the fail2ban-client command:

```
grader@vm:~$ sudo fail2ban-client status
```

You should see a list of all of the jails you enabled:
```
Status
|- Number of jail:	1
`- Jail list:		ssh
```

You can see that fail2ban has modified your firewall rules to create a framework for banning clients. 
Even with no previous firewall rules, you would now have a framework enabled that allows fail2ban to 
selectively ban clients by adding them to purpose-built chains:

```
grader@vm:~$ sudo iptables -S
-P INPUT DROP
-P FORWARD DROP
-P OUTPUT ACCEPT
-N fail2ban-ssh
-N ufw-after-forward
-N ufw-after-input
...
```

You can test the policies. Keep a shell open on the VM in case you need to unban your IP. 
Attempt some bogus ssh logins.

If you look at the status with the fail2ban-client command, you will see your IP address being banned from the site:

```
grader@vm:~$ sudo fail2ban-client status ssh
```

When you are satisfied that your rules are working, you can manually un-ban your IP address with the fail2ban-client by typing:

```
grader@vm:~$ sudo fail2ban-client set apache unbanip 111.111.111.111
```
