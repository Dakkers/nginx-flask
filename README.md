# Nginx & Flask walkthrough

This README will go through the following:
- Setting up Flask
- Setting up Nginx for one domain
- Combining Flask with Nginx for one domain
- Setting up Nginx for multiple domains
- Combining Flask with Nginx for multiple domains

This is assuming you're using Linux. If you want to only do something with one domain, don't bother with multiple domains because the set up is completely different. This also holds if you want to set up multiple domains; do not bother with setting up one domain, because it isn't simply a copy-paste kinda thing.


## Setting up Flask

This is easy. Simply use pip. If you don't have pip installed,
```bash
$ sudo apt-get install python-pip
```

and then

```bash
$ sudo pip install flask
```

Done! If you get some error about some C not compiling, try installing the python-dev package through sudo apt-get.


## Setting up Nginx for one domain

If you are using Apache currently, let's just get this over with:

```bash
$ sudo service apache2 stop
```

(Your website should be down now). Let's install Nginx and start it:

```bash
$ sudo apt-get install nginx
$ sudo service nginx start
```

Then create a directory for your new domain

```bash
$ sudo mkdir -p /var/www/domain
```

(add some html in there yourself). Then make and edit the following file through nano

```bash
$ sudo nano /etc/nginx/sites-available/domain
```

and enter in the following

```
server {
	listen	80;
	root	/var/www/domain;
	index	index.html index.htm;
	server_name domain.com www.domain.com

	location / {
		try_files	$uri $uri/ /index.html;
	}
}
```

and replace the root, index, and server_name as required. If your website is pointed to your server IP (as it should be), we enable the site and restart nginx,

```bash
$ sudo cp /etc/nginx/sites-available/domain /etc/nginx/sites-enabled/domain
$ sudo service nginx restart
```

and it should be working!



## Combining Flask with Nginx for one domain

This is considerably trickier, and more things can go wrong. We first need to install uWSGI, which will allow us to deploy Flask on Nginx, and we should also install screen to allow us to have multiple terminals while ssh'd:

```bash
$ sudo apt-get install build-essential python-dev screen
$ sudo pip install uwsgi
```

Screen creates a nested terminal which can be detached from, which allows simple concurrency. Then, let's make a simple Flask application for our website:

```Python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
	return "Hello from Flask!\n"

if __name__ == '__main__':
	app.run()
```

I'll name it domainscript.py; name it whatever your domain is with the .py extension and then save it in your /var/www/domain folder. To ensure this alone is working, run

```bash
$ screen -S domain
$ cd /var/www/domain
$ python domainscript.py
```

and press Ctrl+A, Ctrl+D. This creates a screen with the title 'domain', runs the Python script inside that screen, and then pressing Ctrl+A and Ctrl+D detaches us from the screen. Then, run

```bash
$ curl 127.0.0.1:5000
```

and "Hello from Flask!" should be the output. Awesome. What we want to do now is edit our Nginx file for our domain such that it recognizes the Python script. To do this, let us simply edit the file:

```bash
$ sudo nano /etc/nginx/sites-available/domain
```

and replace ALL of its contents with the following:

```
server {
	listen	80;
	server_name domain.com www.domain.com;

	location / { try_files $uri @domainscript; }
	location @domainscript {
		include uwsgi_params;
		uwsgi_pass unix:/tmp/uwsgi.sock;
	}
}
```

where @domainscript is the name of your Python script - you MUST include the @ and don't bother with the extension. The uwsgi_pass  thing is the socket we'll create in a moment. Try restarting Nginx,

```bash
$ sudo service nginx restart
```

and then curl your domain - you should get a "502 Bad Gateway" error. Good. Now, edit the last line of your Python script so it instead says

```Python
app.run(host='0.0.0.0')
```

and then run

```bash
$ screen -r domain
$ uwsgi -s /tmp/uwsgi.sock -w domainscript:app --chmod-socket=666
```

(replacing domainscript with the name of your Python script), then detach from the screen (Ctrl+A, Ctrl+D), and restart Nginx, and hopefully it should work.




## Setting up Nginx for multiple domains

If you've used Apache before, you're probably used to the term "virtual host" when hosting multiple domains on one server. With Nginx, we call them "server blocks". The way this is set up is a bit different. If you already made one domain, you can remove the files we made (/etc/nginx/sites-available/domain, /etc/nginx/sites-enabled/domain) and edit a default configuration file:

```bash
$ sudo rm /etc/nginx/sites-available/domain /etc/nginx/sites-enabled/domain
$ sudo nano /etc/nginx/conf.d/default.conf
```

and enter in the following:

```
server {
	listen	80;
	root	/var/www/domainA;
	index	index.html index.htm;
	server	domainA.com www.domainA.com;

	location / {
		try_files	$uri $uri/ /index.html;
	}
}

server {
	listen	80;
	root 	/var/www/domainB;
	index	index.html index.htm;
	server	domainB.com www.domainB.com;

	location / {
		try_files	$uri $uri/ /index.html;
	}
}
```

then restart Nginx, 

```bash
$ sudo service nginx restart
```

and things should be working!



## Combining Flask with Nginx for multiple domains

This is (also) considerably trickier, and more things can go wrong. We first need to install uWSGI, which will allow us to deploy Flask on Nginx, and we should also install screen to allow us to have multiple terminals while ssh'd:

```bash
$ sudo apt-get install build-essential python-dev screen
$ sudo pip install uwsgi
```

Screen creates a nested terminal which can be detached from, which allows simple concurrency. Then, let's make a simple Flask application for our first website:

```Python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
	return "This is Domain A from Flask!\n"

if __name__ == '__main__':
	app.run()
```

I'll name it domainAscript.py; name it whatever your domain is with the .py extension and then save it in your /var/www/domainA folder. Then for our second website:

```Python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
	return "This is Domain B from Flask!\n"

if __name__ == '__main__':
	app.run()
```

I'll name this one domainBscript.py. Again, name yours something with the .py extension and save it in your /var/www/domainB folder. If you want to test if either of these are working, (I'll test domainA), then run:


```bash
$ screen -S domainA
$ cd /var/www/domainA
$ python domainAscript.py
```

and press Ctrl+A, Ctrl+D. This creates a screen with the name domainA, runs the script, and the key combinations detach from the screen. Then run:

```bash
$ curl 127.0.0.1:5000
```

and "This is Domain A from Flask!" should be the output. Then we want to edit our default configuration file again (/etc/nginx/conf.d/default.conf) and replace the content with:

```
server {
	listen	80;
	server_name domainA.com www.domainA.com;

	location / { try_files $uri @domainAscript; }
	location @domainAscript {
		include uwsgi_params;
		uwsgi_pass unix:/tmp/uwsgi_domainA.sock;
	}
}

server {
	listen	80;
	server_name domainB.com www.domainB.com;

	location / { try_files $uri @domainBscript; }
	location @domainBscript {
		include uwsgi_params;
		uwsgi_pass unix:/tmp/uwsgi_domainB.sock;
	}
}
```

where @domainAscript is the name of your Python script for domainA and @domainBscript is the name of your Python script for domainB - you MUST include the @ and don't bother with the extension. The uwsgi_pass  thing is the socket we'll create in a moment. Try restarting Nginx,

```bash
$ sudo service nginx restart
```

and then curl either domain - you should get a "502 Bad Gateway" error. Good. Now, edit the last line of BOTH of your Python scripts so it instead says

```Python
app.run(host='0.0.0.0')
```

and then, if you already created a screen with the name domainA,

```bash
$ screen -r domainA
$ uwsgi -s /tmp/uwsgi_domainA.sock -w domainAscript:app --chmod-socket=666
```

(replacing domainsAcript with the name of your Python script). If you haven't created the screen, follow these steps:

```bash
$ screen -S domainA
$ uwsgi -s /tmp/uwsgi_domainA.sock -w domainAscript:app --chmod-socket=666
```

then detach from the screen (Ctrl+A, Ctrl+D), and do the same for domainB,

```bash
$ screen -S domainB
$ uwsgi -s /tmp/uwsgi_domainB.sock -w domainBscript:app --chmod-socket=667
```

(notice the different chmod-socket number), then restart Nginx, and hopefully it should work.