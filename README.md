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

I configured `ufw` to allow SSH connections on port 2200, and I configured
`unattended-upgrades` to automatically install updates.

I also installed `fail2ban` to block repeated attacks on SSH.

I added two users, `grader` and `helmuth`. The user `helmuth`
is used to actually run the application.
I disabled root login and have added grader the right to sudo with password.

While my application is using sqlite (at the time of Project 3 I never intended
to host it as a production application) I have still installed `postgresql`
together with a database `helmuth` and a user `helmuth`.

I also installed `monit` to allow monitoring. It is listening on port `2812` on
`localhost` - to access the website you have to establish port forwarding. The
password is in the config file /etc/monit/monitrc which is only readibly by root.

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

And here is the Apache configuration for the application:
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