![uwsgi_nginx_python](https://s3-ap-southeast-2.amazonaws.com/innablr/uwsgi_nginx_python.png)

### Setting Up uWSGI

Before you start using NGinx you'll need to run an update on your version of Ubuntu.

```
sudo apt-get update
sudo apt-get install python-dev python-pip nginx
```

Next, setup your virtual environment:

```
sudo pip install virtualenv
source appenv/bin/activate
```

After completing the above and setting up your virtual env, we can start configuring NGinx. Any packages we install here will only exist inside the virtual environment. No we can install uWSGI into the environment:

```
pip install uwsgi
```

Verify the installation by typing:

```
uwsgi --version
```

Now we can create the wsgi.py application or entry point:

```
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return ["<h1 style='color:blue'>Hello There!</h1>"]
```

uWSGI server looks for a callable function called 'application' by default. Now we can test, run the following command to start the WSGI server:

```
uwsgi --socket 0.0.0.0:8080 --protocol=http -w wsgi
```

Now visit the servers IP (I am running this instance on EC2 - make sure you allow port 8080 in the security group or else you wont be able to access the page) and verify access. In the above, we manually provided the command to run the server and pass in our parameters from the command line. Like terraform, we can (somewhat) automate this process using a configuration file:

```
nano ~/python-microservices/building_microservices/app.ini
```

Inside our config file we create a section [uwsgi] and point it to our application:

```
[uwsgi]
module = wsgi:application
```

The first process run on the wsgi server will be the 'master process', all other processes run after this will be worker processes.

```
master = true
processes = 5
```

In my command that I used to run the uWSGI server, I specified that it use the http protocol. We are going to install NGinx in front of uWSGI (Nginx -> uWSGI -> Microservice) and as a result, change the protocol in use. NGinx comes with a uWSGI mechanism out of the box. By ommitting any protocol specficiations in the uWSGI command, we actually force uWSGI to fallback to the uWSGI protocol as a default. Also, since we are using this config with NGinx on the same machine, we can configure a socket instead of a port. This will yield faster performance results. Ill assign the socket permissions 664 (User and Read/Write access, group read access). I'll also use the vacuum option - allowing the socket to be released again once the process has stopped.

```
socket = myapp.sock
chmod-socket = 664
vacuum = true
```

Finally, we are going to start the application at boot with an upstart file. Upstart and uWSGI both use SIGTERM but for different things. To get around this we add an option called 'die-on-term' to stop uWSGI from restarting the program unexpectedly.

So altogether my configuration file looks like this:

```
master = true
processes = 5

socket = myapp.sock
chmod-socket = 664
vacuum = true

die-on-term = true
```

With that out of the way, we can create an upstart script to launch a uWSGI application at boot time, making sure our application is always available.

```
sudo nano /etc/init/myapp.conf
```

We can start with a description and system run levels.

```
description "uWSGI instance to serve myapp"

start on runlevel [2345]
stop on runlevel [!2345]
```

Now we can specify user information to run the process as. We will also specify 'www-data' (this is a user account in Nginx) so that Nginx can talk to the socket defined in our .ini file.

```
setuid demo
setgid www-data
```

Next, we'll use a script block to activate uWSGI inside the virtual environment. This makes it easier to access software which is also configured in the virtual environment.

```
script
    cd /home/demo/myapp
    . myappenv/bin/activate
    uwsgi --ini myapp.ini
end script
```

The finished product should look like this(substitute your own username obviously):

```
description "uWSGI instance to serve myapp"

start on runlevel [2345]
stop on runlevel [!2345]

setuid demo
setgid www-data

script
    cd /home/demo/myapp
    . myappenv/bin/activate
    uwsgi --ini myapp.ini
end script
```

We should now be able to start the service by just typing

```
sudo start myapp
```


### Setting Up Nginx

We are going to configure NGinx as a 'reverse proxy' basically, it will take http requests and redirect them to the uWSGI socket and then eventually on to our application. Navigate to the following directory and create a file called 'myapp' or whatever you prefer to call it. This is our conmfiguration file for Nginx. In this case it will be very rudimentary:

```
server {
    listen 80;
    server_name server_domain_or_IP;

    location / {
      include         uwsgi_params;
      uwsgi_pass      unix:/home/demo/myapp/myapp.sock;
  }
}
```

The location block that begins with '/' in the above config, basically routes all traffic to the same place (uwsgi --> myapp). Link the config file to the available-sites with the following command:

```
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled
```

![app_available](https://s3-ap-southeast-2.amazonaws.com/innablr/app_available.PNG)
