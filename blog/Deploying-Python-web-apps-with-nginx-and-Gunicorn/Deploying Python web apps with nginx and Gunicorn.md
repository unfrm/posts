# Deploying Python web apps with nginx and Gunicorn

First of all, I should mention that all the information inside this article is suitable mostly for small projects. Serving large and heavy loaded apps is a completely different topic, which is, speaking honestly, goes beyond my knowledge and skills. So if you're just a Python hobbyist and you're looking a way to put your apps into the Web, I bet these next few words can be helpful.

So here's the deal. Basically there are a few global approaches to make your Python apps visible for everyone:

- PaaS (platform-as-a-service, <a href="https://www.pythonanywhere.com" target="_blank">PythonAnywhere</a>, <a href="https://developers.openshift.com" target="_blank">OpenShift</a>, <a href="https://www.heroku.com/" target="_blank">Heroku</a>) - provides its own software level above the hardware, meaning that your app will be deployed exactly on that software level.
- Iaas (infrastructure-as-a-service, <a href="http://aws.amazon.com" target="_blank">Amazon AWS</a>) - if you ask me, it works pretty much the same as virtualized servers (I mean the way you get the resources), the only difference is how you pay for it.
- Bare metal servers - it's pretty clear from the title, although I doubt a private person would engage in such.
- Virtualized servers (VPS, there are a tons of providers) - are a virtual piece of resources sliced from a larger, physically existing servers.

I'm not going to speak about pros and cons, instead, here are a few reasons why I didn't choose the first option, which is way more popular than others, and why I turned my attention on a VPS.

**Flexibility and customization.** I came to the conclusion that it's not about the PaaS platform. All your actions are limited to that software layer, on which your apps are working. No doubt, this approach can be flexible enough to maintain really huge apps, but along with that, I always want to manage everything personally, which is not possible because of the way how you get the resources.

**Fun.** That's right, it can be fun enough. You won't get it with PaaS. By using a VPS you're starting with a clean slate, and you have to do a lot of different things, believe me, it's much more interesting than doing everything in one minute (of course, this may not be applicable for commercial projects).

**Pricing.** I don't like the pricing for PaaS platforms, it is unreasonably high when we talk about some serious load (personal apps can also be heavy, yeah). And this is doubly pointless when you take into account the fact that you can get a VPS in <a href="http://aws.amazon.com" target="_blank">Amazon AWS</a> for free. Yep, it's only for one year, but still.

However, upon the rejection of the PaaS deployment method, you may encounter a series of problems with support, maintenance, and, speaking honestly, it won't that fast as it can be with <a href="https://www.pythonanywhere.com" target="_blank">PythonAnywhere</a>. So if you don't afraid to do everything by self, then my experience will be useful to you. Otherwise, feel free to use the links above and read more about different PaaS platforms, most probably they will fit your needs by 100% (and yes, free plans are also included).

As for me, I use <a href="https://www.arubacloud.com" target="_blank">Aruba Cloud</a> for my personal needs, but for this article I will use a VPS hosted in <a href="http://aws.amazon.com" target="_blank">Amazon AWS</a>. Although it has no special significance, since in the end, we only work with the operating system that will be installed on our VPS. So you can choose any provider you like, or you can do this even locally.

# Prerequisites

Before we start, let's be sure we have everything we needed.

**VPS.** As you might guess, our VPS will be running on Linux. It doesn't mean you can't deploy your Python WSGI apps using IIS server under Microsoft Windows, actually you <a href="https://pypi.python.org/pypi/PyISAPIe" target="_blank">can</a>, but let's be honest - this is damn unusual way. So we'll be more traditional.

If you ask me, I'm a big Debian fan, however, this is not some kind of dogma, you can use whatever you like, just keep in mind the difference between deb- and rpm-based distributions.

So once we got a VPS (in this article it will be running on <a href="https://www.debian.org/releases/jessie/" target="_blank">Debian 8</a>), let's be sure the local index of packages is up to date. In order to do that, proceed with these two simple commands:
> Ignore the "sudo" part if you're logged in as root, but I strongly recommend you to create another user and delegate it all needed permissions - it is **bad** to login as root.

```
sudo apt-get update
```

```
sudo apt-get upgrade
```

**Python.** After that, let's check if we have Python installed (most likely we have, it comes by defaults in most distributions). Just type the command below:

```
python
```

You should be able to see something like:

```
Python 2.7.9 (default, Mar  1 2015, 12:57:24)
[GCC 4.9.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
```

**Nginx.** Now we're ready to do a little of magic. Type the following command in order to install our nginx server:

> No need to choose any specific version, let's stick to the version that is the default version in your repository (mine is 1.6.2, however the mainline is <a href="http://nginx.org/en/download.html" target="_blank">1.11.2</a>).

```
sudo apt-get install nginx
```

Let's start our web server:

> If you using Amazon AWS, don't forget to check the Security Groups and allow HTTP/HTTPS protocols in your Inbound tab.

```
sudo service nginx start
```

Then you can type this one, to check if the service was up and running:

> Anyway, if you have any problems, like wrong configuration, etc., you'll be notified during the execution of previous command.

```
sudo service --status-all | grep nginx
```

And, finally, open the browser and go to the address of your server. If everything is OK, you will see something like this:

> In case of doing this locally, it should be http://localhost/.

<img src="http://unfrm.us/static/img/posts/20160718/nginx_started.png" alt="nginx started">

Okay, it works, we can stop it (at least for now):

```
sudo service nginx stop
```

**Gunicorn.** Last but not least - the Green Unicorn, a Python WSGI HTTP server. There is two different ways to install it. If you want to use it as a Python module, or if you want to use a virtualenv, you can install it directly to your regular/isolated Python environment (I hope you know what a <a href="https://virtualenv.pypa.io/en/stable/" target="_blank">virtualenv</a> is) by using a simple pip command:

> Here you can use the latest available version - <a href="http://docs.gunicorn.org/en/stable/news.html#id1" target="_blank">19.6.0</a>.

```
pip install gunicorn
```

Or, if you want to use a virtualenv:

```
/path/to/your/virtualenv/bin/pip install gunicorn
```

Also, you can use a system package available from the repository. To do this, just type the following:

```
sudo apt-get install gunicorn
```

I suggest you to install it as a system package, because it will be easier.

If the installation was OK (and I'm sure it was), we're ready to move forward.

# A few words about how this should work

It is likely that by this point you already got a question - why do we even need nginx? Gunicorn is a server, too! But it's not as easy as it seems at first glance.

That's why we use nginx as reverse proxy to our Gunicorn instance. Meaning that nginx will have a couple directives in its config, to rewrite several headers and to redirect all requests to our Gunicorn server running on the local host. I know, I know, it's not clear enough, but you will get it once we'll start configuring our HTTP proxy. Going back to the question "why do we even need this":

- It is more secure.
- It is much more faster.
- Nginx is more suitable to serve static content.
- It is more flexible, with that we can configure our app a much more subtle way.

And now we get to the very essence of our adventure.

# Configuring nginx

A few lines above we made sure that our nginx server was up and running. There was no attention to its configuring, simply because it works just from the box. But now we are going to сhange some parts of the default config. I will try to describe the whole process as detailed as possible, but please keep in mind that we are not creating perfect nginx configuration, so I'm not going to pay attention on those things that do not relate to the topic of this article (like security issues, performance tuning, and so on). In the end, luckily for us nginx is a well documented project, so I believe it's not that hard to find out how to configure it properly just using the official docs.

First of all, let's talk about how it uses its config. Here's the basics.

The primary one is **nginx.conf**. You are free to edit it whatever you like, but I suggest you to forget this way. You still have to keep some global preferences within this file, but all that relates solely to your app will be located in a different place.

There is a wonderful feature that allow us to split configuration for each individual site. This is usually used in order to have multiple virtual hosts on a single server, but it is also a great way to make our server even more flexible. It makes the server configuration more convenient, and minimizes the number of possible errors. This is achieved by using the **include** statement. Now open nginx.conf with your favourite editor (mine is <a href="https://www.nano-editor.org" target="_blank">nano</a>):

> Please remember that depending on the system or the installation method, the location of this file may be different.

```
sudo nano /etc/nginx/nginx.conf
```

Scroll down to Virtual Host Configs:

	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;

As you can see there is an include statement, it tells nginx to load all configurations located within that locations. This is a really cool way to keep the basic preferences in nginx.conf, and to use site-enabled for everything else, in cases when it should relate to specific sites rather than the global environment. As for the **sites-enabled** itself, it is a directory that contains a symlinks to another directory called **sites-available**. I bet you've guessed what logic is used here! All your sites are in site-available. As soon as your app goes on production and you want to make it visible for everyone, just make a symlink to its config, then put it to sites-enabled - voila, everything works!

Now let's create a new config inside sites-available:

```
sudo nano /etc/nginx/sites-available/testapp.conf
```

Put inside it the following text:

	upstream testapp {
		server 127.0.0.1:8000;
	}

	server {
		listen 80;
		server_name www.example.com;

		root /var/www;
		index index.html;

		location / {
			try_files $uri @proxy;
		}

		location @proxy {
			proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
			proxy_set_header Host $http_host;
			proxy_set_header X-Scheme $scheme;
			proxy_redirect off;
			proxy_pass http://testapp;
		}
	}

This is the basic configuration for our app and it is easy enough to understand how it works, so let's take a look on it. **Upstream** is a directive, it just defines a server, in our case we specify the local host, which will run our Gunicorn instance. Then, **server** sets configuration for a server, we listening a port number 80 (that is the default one), we also define a server name by using the appropriate directive. After that, we define a root path (by default, this was /var/www/html) and type of files that can be used as an index. Next, we set a **location** directive, which works as follows: we send a request to www.example.com/ → nginx is trying to find an index file in our root location (that's what **try_files** does) → if nothing was found, the request is redirected to our **@proxy**.

So the **@proxy** part is most important here, it allows our app to work correctly. The first three entries modify the headers the right way, the next one is used to prevent nginx from any redirects, and, finally, the last one is passing our request to the local host, which, as you remember, was defined early.

> If you want to see your app running on path that is different from **/**, just edit the first location directive to, let's say, **location /testapp**, then, add the following statement to your **@proxy** part - **proxy_set_header X-Script_name /testapp;**. Now your app will be possible to run under www.example.com/testapp.

> The attentive reader may have noticed that we didn't specify the path to save logs. This is because we use the paths that are specified by default in nginx.conf. Feel free to do it another way, especially if you plan to have several virtual hosts.

Okay, we're almost there. Now let's add a symlink, so that nginx can see our configuration:

```
sudo ln -s /etc/nginx/sites-available/testapp.conf /etc/nginx/sites-enabled
```

And don't forget to remove a symlink called **default** (it was default settings for our newly installed nginx):

```
sudo rm /etc/nginx/sites-enabled/default
```

Start our server, and now we are ready to move on to the last part:

```
sudo service nginx start
```

# Configuring Gunicorn

Basically, Gunicorn can be configured by using the following methods - through a command line, configuration file, or by using a framework settings. I prefer to use a separate config, but for this article we'll use a command line configuration.

What I love about this server is that it works with minimal config. We are interested in only these two parameter: **-w** is for workers serving our app, **-b** is for binding proper address to work on. Let's move on to see how it should be used.

# It's time to put it all together

As you remember, we already defined a root path for our project. Now let's create the app itself. Open a text editor and type this:

	def testapp(environ, start_response):
		data = 'Hello, world!\n'
		status = '200 OK'
		response_headers = [
			('Content-type','text/plain'),
			('Content-length',str(len(data)))
		]
		start_response(status, response_headers)
		return iter([data])

Save it as **testapp.py** in /var/www, go to /var/www and try to start it (no need to use sudo here):

```
gunicorn -w 2 -b 127.0.0.1:8000 testapp:testapp
```

You may see something similar to this:

```
[INFO] Starting gunicorn 19.6.0
[INFO] Listening at: http://127.0.0.1:8000 (1843)
[INFO] Using worker: sync
[INFO] Booting worker with pid: 1848
[INFO] Booting worker with pid: 1849
```

That means we launched a gunicorn instance with 2 workers running on binded 127.0.0.1:8000. Now open the browser and go to the address of your server:

<img src="http://unfrm.us/static/img/posts/20160718/gunicorn_started.png" alt="gunicorn started">

Congratulations, we're almost done!

# A few finishing touches

You might noticed that launching our Gunicorn instance as an active job is not something that is very convenient to use. From this point we have two options - we can either daemonize the Gunicorn process, or we can use a separate monitoring tools.

Daemon-izing a process is easy, all you have to do is just to use a **-D** key with your starting parameters. With that your server will be detached from the terminal and entered the background. However, you have to repeat it every time after rebooting your server. Let's try something more flexible.

I prefer to use Gunicorn with <a href="http://smarden.org/runit/" target="_blank">runit</a>. To create a new configuration file for our test app, let's run our favourite text editor:

	#!/bin/sh

	GUNICORN=/usr/local/bin/gunicorn
	ROOT=/var/www
	PID=/var/run/gunicorn.pid
	APP=testapp:testapp

    if [ -f $PID]; then rm $PID; fi

	cd $ROOT
	exec $GUNICORN -w 2 -b 127.0.0.1:8000 --pid=$PID $APP

The syntax is simple, we just defined a couple of variables, an if statement that checks if the PID file already exists, and then, we run our Gunicorn instance, nothing more. Save it as **run** in **/etc/sv/testapp** (you need to create this directory before saving the script), and make it executable:

```
sudo chmod u+x /etc/sv/testapp/run
```

Now let's create a symlink so that runit can start it properly:

```
sudo ln -s /etc/sv/testapp /etc/service/testapp
```

After that, the server should start automatically. Type this to re-check it:

```
ps aux | grep testapp
```

The output should be similar to this, a master process, 2 workers and a runit daemon:

```
root       305  0.0  0.0   4100   684 ?        Ss   13:17   0:00 runsv testapp
root       491  1.0  1.5  55788 16204 ?        S    15:06   0:00 /usr/bin/python /usr/local/bin/gunicorn -w 2 -b 127.0.0.1:8000 --pid=/var/run/gunicorn.pid testapp:testapp
root       496  0.0  1.2  55788 12844 ?        S    15:06   0:00 /usr/bin/python /usr/local/bin/gunicorn -w 2 -b 127.0.0.1:8000 --pid=/var/run/gunicorn.pid testapp:testapp
root       497  0.0  1.2  55788 12844 ?        S    15:06   0:00 /usr/bin/python /usr/local/bin/gunicorn -w 2 -b 127.0.0.1:8000 --pid=/var/run/gunicorn.pid testapp:testapp

```

Reboot your server, then open the browser and go to the address of your server:

<img src="http://unfrm.us/static/img/posts/20160718/gunicorn_started.png" alt="gunicorn started">

By the way, you can control it in this way:

```
sudo sv start|restart|stop testapp
```

> In some cases you have to do a similar job for nginx, however, usually there is no need in doing this.

That's all, our VPS is ready to work. Check the links below to learn about more advanced ways to configure the components that we used in this article.

See ya!

# Related links

- <a href="http://nginx.org/en/docs/" target="_blank">Nginx documentation</a>
- <a href="http://docs.gunicorn.org/en/stable/" target="_blank">Gunicorn documentation</a>
- <a href="http://smarden.org/runit/" target="_blank">A few words about runit</a>
- <a href="http://aws.amazon.com" target="_blank">Get your own VPS for free</a>
