# fullstack-nanodegree-webserver

### 1./2. Launch VM via Udacity Account and SSH into VM

```
me@local:~$ ssh -i ~/.ssh/udacity_key.rsa root@52.37.55.67
```

### 3. Create new user named grader

```
root@vm:~$ adduser grader 
``` 

### 4. Give grader permission to sudo

```
root@vm:~$ vi /etc/sudoers.d/grader
```

Add these lines to file grader:

**/etc/sudoers.d/grader**
```
grader ALL=(ALL) NOPASSWD:ALL`
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
Open the default page in a browser by going to `http://52.37.55.67/`.

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
Open your browser at `http://52.37.55.67/` and see `Hello World` welcome.

#### Clone fullstack nanodegree project 3

```
grader@vm:~$ sudo apt-get install git
grader@vm:/var/www$ sudo git clone https://github.com/cg94301/fullstack-nanodegree-catalog.git
grader@vm:/var/www$ sudo ln -s fullstack-nanodegree-catalog/vagrant/catalog
```

#### Install dependencies for project 3

```
grader@vm:~$ sudo apt-get install python-pip
grader@vm:~$ sudo pip install Flask
grader@vm:~$ sudo pip install SQLAlchemy
grader@vm:~$ sudo pip install oauth2client
```

You need client_secrets.json. Use scp to copy to VM:

```
me@local:~$ scp -P 2200 -i ~/.ssh/udacity_key.rsa client_secrets.json grader@52.37.55.67:
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
from application import app
app.secret_key = 'catalog_secret_key'
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
        </Directory>
        Alias /static /var/www/catalog/static
        <Directory /var/www/catalog/static>
            Order allow,deny
            Allow from all
        </Directory>
</VirtualHost>
```

```
grader@vm:~$ sudo a2dissite 000-default
grader@vm:~$ sudo a2ensite catalog
grader@vm:~$ sudo service apache2 reload
```
