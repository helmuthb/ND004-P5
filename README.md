# Full Stack Web Developer Nanodegree - P5: Linux Server Configuration

This is the project 5 implementation for the Full Stack Web Developer
Nanodegree, configuring a Linux server (Ubuntu) to run an Flask-based
application

## Prerequisites

The Server IP address is `52.24.129.145`.
It has a public DNS A-Record: `ec2-52-24-129-145.us-west-2.compute.amazonaws.com`.

## Running the website

To use the web site it is mandatory to use the name, as Google Sign In does not
work with IP addresses.
Therefore access the application via
http://ec2-52-24-129-145.us-west-2.compute.amazonaws.com

## Configuration

I have install Apache with mod_wsgi, postgresql, and the required python modules:
```
apt-get -qqy install postgresql python-psycopg2`
apt-get -qqy install python-flask python-sqlalchemy
apt-get -qqy install python-flask-sqlalchemy
apt-get -qqy install python-pip python-markdown
pip install bleach
pip install oauth2client
pip install requests
pip install httplib2
pip install Flask-Markdown
```

SSH has been configured to use port 2200 instead of the default 22
through changes in the file `/etc/ssh/sshd_config`, and root login
has been disabled as well:
```
...
# What ports, IPs and protocols we listen for
# Changed to port 2200 instead of default 22
Port 2200
...
# PermitRootLogin without-password
# I disabled root login - only user grader can sudo
PermitRootLogin no
...
```

I configured `ufw` to allow SSH connections on port 2200, and I configured
`unattended-upgrades` to automatically install updates.
```
ufw enable
ufw allow 2200/tcp
ufw allow 80/tcp
ufw allow 123/udp

dpkg-reconfigure unattended-upgrades
```

I also installed `fail2ban` to block repeated attacks on SSH:
```
apt-get install fail2ban
cd /etc/fail2ban
cp jail.conf jail.local
vi jail.local # update port for SSH from 22 to 2200
```

I added two users, `grader` and `helmuth`. The user `helmuth`
is used to actually run the application.
The password for the `grader` was generated using
[strongpasswordgenerator.com], 15 characters, avoiding similar characters and
avoiding programming punctuation.
The password for `helmuth` is disabled in `/etc/shadow`.
For both users the file `$HOME/.ssh/authorized_keys` has been created with the
public key used to access the machine.

I disabled `root` login and have added `grader` the right to sudo with password
using `visudo`:
```
...
# Give grader access to root
grader ALL=(ALL) ALL
...
```

While my application is using sqlite (at the time of Project 3 I never intended
to host it as a production application) I have still installed `postgresql`
together with a database `helmuth` and a user `helmuth`.

I also installed `monit` to allow monitoring. It is listening on port `2812` on
`localhost` - to access the website you have to establish port forwarding. The
password is in the config file `/etc/monit/monitrc` which is only readibly by root.
```
apt-get install monit
vi /etc/monit/monitrc
```

Some adjustments were necessary on the appliction to make it run with `mod_wsgi`.
Some of the initialization code was within the `main` block, I had to move it
out so it is also executed for wsgi:
```
# check if db file exists - if not create it
if not os.path.isfile('catalog.db'):
    with app.test_request_context():
        db.create_all()
# set debugging
app.debug = True
# create key for session storage
app.secret_key = getRandomString()
# assign my json encoder
app.json_encoder = MyJSONEncoder
# add markdown filter
Markdown(app)

# when this script is called from the command line
# vs being imported as a module from some other module
if __name__ == '__main__':
    # start the Flask server on port 8080
    app.run(host='0.0.0.0', port=8080)
```

The `wsgi` script I use looks like this:
```
import sys
import os
sys.path.insert(0, '/home/helmuth/ND004-P3/vagrant/catalog')
os.chdir('/home/helmuth/ND004-P3/vagrant/catalog')
from application import app as application
```

In Apache the default website has been disabled (`a2dissite 000-default`) and a new
config file for the wsgi-based site has been added and enabled
(`vi /etc/apache2/sites-available/wsgi.conf`, `a2ensite wsgi`):
```
<VirtualHost *>
    # specify the server name - Google OAuth only works with a real name,
    # not an IP address
    ServerName ec2-52-24-129-145.us-west-2.compute.amazonaws.com

    # run the WSGI process as user helmuth
    WSGIDaemonProcess nd004-p3 user=helmuth group=helmuth threads=5
    # link to the .wsgi python script which defines the application
    WSGIScriptAlias / /var/www/nd004-p3/nd004-p3.wsgi

    # standard WSGI configuration - taken from Flask documentation
    <Directory /var/www/nd004-p3>
        WSGIProcessGroup nd004-p3
        WSGIApplicationGroup %{GLOBAL}
        Order deny,allow
        Allow from all
    </Directory>
</VirtualHost>
```

## Copyright and license

My contributions are in the public domain.