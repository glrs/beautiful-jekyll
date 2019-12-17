---
layout: post
title: Jupyter notebook for personal online access
tags: guide jupyter-notebook
published: true
---

This is a guide of how to setup a Jupyter Notebook for remote access. Not a novelty whatsoever, just a nice way to work on your projects (especially if you have a strong machine sitting somewhere), while you are wandering around carrying only your incapable laptop. As a ML/DL enthusiast I run TensorFlow in a fairly powerful remote server, so I want to exploit the most out of it, with the convenience of the Jupyter Notebook.

We need:
- an Ubuntu server[*](#notes) with Internet access & sudo privileges
- Apache2 server
- Anaconda

Once everything is set up, we should be able to access the Jupyter Notebook on a web browser through the web.

**Disclaimer:** This demonstration is about a personal, single-user notebook that you will be able to access remotely. This is different from a multi-user server setup. If you want to setup a server for multi-user access you can refer to this [GitHub Repo](https://github.com/jupyterhub/jupyterhub). 

**[Jump straight to the guide! (Let's Start)](#lets-start)**

## Why Apache?
You probably know that Jupyter Notebook comes with its own server, so why not using this server straight away instead of Apache (or any other server)? Well, there are some reasons not to do so. Most of the times your server machine is behind a firewall (and if not, it should be) and this makes it tricky, tedious and insecure to open new ports, especially if you are not very familiar with networking and security. One could say that you can use already opened ports, like port 80. However, if you want to run a web server simultaneously (which I usually do) you have to reconsider it. Also, Apache is on the market for many-many years, maintaining a good reputation as web servers are concerned. Thus, I trust it way more than I would trust a recent server intended to serve the Jupyter Notebook on localhost. For sure they are doing a very good job in optimizing the security on Jupyter server, as you can read [here](http://jupyter-notebook.readthedocs.io/en/latest/security.html) …but still.

So, why Apache and not another famous server like NGINX? This is up to you! Apache is my own server of choice, because I am more familiar with it and I want to do things faster. But as I said, you should feel free to use the web server of your choice. Here I am using Apache2 to proxy the Jupyter Notebook server.

## Why Anaconda?
Anaconda packs together many goodies (Python, Jupyter Notebook, conda, etc) and also prevents the user from messing too much up with the system. You can read more about Anaconda’s advantages it in this [reddit thread](https://www.reddit.com/r/Python/comments/3t23vv/what_advantages_are_there_of_using_anaconda/).


## Let's Start!
### Installations[*](#notes)
Open your terminal (or ssh session) and let’s get started!

First of all, make sure that your system is updated
```shell
sudo apt-get update		# Fetches the list of available updates
sudo apt-get upgrade		# Strictly upgrades the current packages
sudo apt-get dist-upgrade	# Installs updates (new ones)
sudo reboot			# Restart
sudo apt-get autoremove		# Remove packages system no longer needs
```

For upgrading from an older Ubuntu version (recommended) do the following.
WARNING! Before you proceed backup your data and/or read this [askubuntu thread](http://askubuntu.com/questions/110477/how-do-i-upgrade-to-a-newer-version-of-ubuntu).

(If you are connected via SSH, do the upgrade in a TMUX session: `tmux new –s upgrade_session`)

Proceed with the upgrade
```shell
sudo do-release-upgrade
```

If for some reason after the distro upgrade you see a message like:

`N: Ignoring file ‘50unattended-upgrades.ucf-old’ in directory ‘/etc/apt/apt.conf.d/’ as it has an invalid filename extension`

then simply remove the file.
```shell
sudo rm /etc/apt/apt.conf.d/50unattended-upgrades.ucf-old
```
Read about this issue in this [askubuntu thread](http://askubuntu.com/questions/829370/n-ignoring-file-50unattended-upgrades-ucf-dist-in-directory-etc-apt-apt-con).

### > Apache
Next step is to install Apache in our system. This should be as easy as:
```shell
sudo apt install apache2
```

To verify that the installation was successful type: `service apache2 status`

```shell
 ● apache2.service - LSB: Apache2 web server
    Loaded: loaded (/etc/init.d/apache2; bad; vendor preset: enabled)
   Drop-In: /lib/systemd/system/apache2.service.d
            └─apache2-systemd.conf
    Active: active (running) since Wed 2016-12-21 15:25:01 UTC; 15min ago
      Docs: man:systemd-sysv-generator(8)
    CGroup: /system.slice/apache2.service
            ├─2558 /usr/sbin/apache2 -k start
            ├─2561 /usr/sbin/apache2 -k start
            └─2562 /usr/sbin/apache2 -k start
Dec 21 15:24:59 test-install systemd[1]: Starting LSB: Apache2 web server...
Dec 21 15:24:59 test-install apache2[2522]:  * Starting Apache httpd web server apache2
Dec 21 15:25:01 test-install apache2[2522]:  * Dec 21 15:25:01 test-install systemd[1]: Started LSB: Apache2 web server.
```

More info for [Apache2 here](https://help.ubuntu.com/lts/serverguide/httpd.html).

### > Anaconda
Then we are ready, to download and install Anaconda. Go to [https://www.continuum.io/downloads](https://www.continuum.io/downloads) and download Anaconda with Python3.6 for your system. Or, as I like to do, find the desired distribution in the [https://repo.continuum.io/archive/](https://repo.continuum.io/archive/) and download with wget (or curl).
```shell
wget https://repo.continuum.io/archive/AnacondaX-X.X.X-Linux-x86_64.sh
```

Then run the script you just downloaded to begin the installation, and follow the instructions (if unsure, just accept the defaults).

NOTE: I recommend you to select adding the install location to PATH when you asked.

```shell
bash AnacondaX-X.X.X-Linux-x86_64.sh
```
For more information about the Anaconda installation, check the [documentation here](https://docs.continuum.io/anaconda/install#linux-install)

Afterwards, you need to close your current terminal and open a new one for the changes to take effect.

To verify that the installation was successful, type:
```shell
jupyter notebook
```
If you get any messages about JavaScript requirements, just ignore it for now.

It is time to install the `setuptools` package using `pip`. To find out which `pip` to use run the following.
```shell
which pip
pip --version
```
This `pip` alias should point to the Anaconda's `pip3` (if you chose the suggested Anaconda with the Python3). If not, then try the above commands typing `pip3` instead of `pip`. This is for you to know which pip to use for installing stuff that are visible by Anaconda.

Now install the latest setuptools (using the appropriate `pip`)
```shell
pip install -U pip setuptools
```

## Configuration
### > Configure Jupyter
Now it's time to configure the Apache and Jupyter Notebook to make them work together. First, we need to create a configuration file for Jupyter
```shell
jupyter notebook --generate-config
```
This should create a folder (named `.jupyter`) in the home directory. This folder contains the configuration file named `jupyter_notebook_config.py`. Open it with your text editor (I use vim) and write the followings at the beginning of the file (you can also uncomment and edit the respective fields already existing in the document).
```shell
vim .jupyter/jupyter_notebook_config.py
```
```conf
# Configuration file for jupyter-notebook.
c = get_config()
c.NotebookApp.ip = 'localhost'
c.NotebookApp.port = 8888  # or other port number of your preference
c.NotebookApp.base_url = '/jupyter'  # or whatever you like to have as url
c.NotebookApp.open_browser = False
```
**Note:** If you got any messages about JavaScript requirements when running `jupyter notebook`,  then the last option (`open_browser = False`) should prevent them. What is happening is that Jupyter by default tries to open a browser (which in my case is impossible since I run a command line only server). By setting this option to False, no browser is invoked, so no more JavaScript warnings will pop up.

What we have done until now, is to configure the Jupyter Notebook to run as a server at localhost on the port 8888, and under the url `http://localhost:8888/jupyter` (or whatever you configured it to). Note that it is not necessary to specify a base_url, however you will need it if you run other stuff (e.g. a web page) under the root folder.

### > Configure Apache
The first step when configuring Apache is to make sure that we can have public access to it. Make sure that the port 80 and 443 (HTTP and HTTPS) of your system are opened and not blocked by the firewall.

If you have a VM server on a high-end Platform (e.g. Google Cloud), then allowing HTTP access should be as simple as checking a box (e.g. “Allow HTTP traffic”). For more general info about opening HTTP ports take a look at [https://www.cyberciti.biz/tips/linux-iptables-11-how-to-block-or-open-httpweb-service.html](https://www.cyberciti.biz/tips/linux-iptables-11-how-to-block-or-open-httpweb-service.html).

By default there should be an `index.html` file in the `/var/www/html/` folder. If not, just create a dummy index file with some text in it. Then try to visit your server outside the local network (use its global IP address). At this point I assume you succeeded to connect, and we move on.

Next step would be to configure Apache to reverse proxy the location we created earlier for the Jupyter Notebook server. Open the Apache’s configuration file `apache2.conf` under `/etc/apache2/` (default) directory. Note that you might need administrative privileges for being able to edit this file.
```shell
sudo vim /etc/apache2/apache2.conf
```
Go at the end of the file and add the following:
```apache
# Allow Jupyter through apache2
<Location /jupyter>
ProxyPass        http://localhost:8888/jupyter
ProxyPassReverse http://localhost:8888/jupyter
RequestHeader set Origin "http://localhost:8888"
</Location>

<Location /jupyter/api/kernels/>
ProxyPass        ws://localhost:8888/jupyter/api/kernels/
ProxyPassReverse ws://localhost:8888/jupyter/api/kernels/
</Location>
```
Of course if you have chosen a different port or a different base URL in the Jupyter config file, you have to change them accordingly to reflect your preferences. On the second part we simply reverse proxying the Jupyter Kernel WebSockets too (it is necessary). For more on this and the Jupyter kernels take a look at [http://jupyter-kernel-gateway.readthedocs.io/en/latest/](http://jupyter-kernel-gateway.readthedocs.io/en/latest/).

The last piece on the puzzle that will make the whole setup work, is to enable some of the Apache modules. Go forward and type the next commands on your terminal.
```shell
sudo a2enmod proxy		# Enable proxy module
sudo a2enmod proxy_http		# Enable proxy http module
sudo a2enmod proxy_wstunnel	# Enable proxy websocket tunnel
sudo a2enmod headers		# Enable headers (for RequestHeader directive)
sudo service apache2 restart	# Restart Apache
```
I figured out most of these settings by reading on this [answer on Stackoverflow](http://stackoverflow.com/a/28819231/4618605).

By the time that Apache restarts you should be able to access your Jupyter Notebook online!

**Recommendation**: Open a new TMUX session to run your Jupyter server in it and give it a relevant name: `tmux new -s JupyterNotebookServerSession`

Start the Jupyter Notebook
```shell
jupyter notebook
```
Now visit your server from a remote browser, (e.g. `http://<your-ip-address>/jupyter`) , et voilà!!! Enjoy your very own notebook from everywhere!

## Security!
Being online means being exposed. Here I will describe only the security basics (a.k.a. how to add password to your notebook), but I will provide some resources on how to upgrade your notebook’s/server's security even further.

First of all, we need to create a hashed password. You can very easily do this by running Python on your terminal (make sure it is the Anaconda installed Python) and typing the following:
```python
>>> from notebook.auth import passwd
>>> passwd()	# You will be prompted to enter your password
    
	Enter password: <YOUR-PASSWORD>
	Verify password: <YOUR-PASSWORD>
	‘sha1:<HASHED-SALT>:<HASHED-PASSWORD>’
```

After typing and verifying your password, a sha1 string which contains your hashed password and the hashed salt will appear. Make sure you keep that somewhere that you can find it later. Now open the Jupyter’s config file and add the next line:
```conf
c.NotebookApp.password = u'sha1:<hashed-salt>:<hashed-password>'
```
Replace the placeholder with the SHA1 string you retained earlier. Save. Attach to your TMUX session (if you use one) and restart your Jupyter Notebook. Refresh your notepad on your browser, and you should be prompt to add your password to login. 1st level of security done!

## More security
Complete Jupyter security documentation: [http://jupyter-notebook.readthedocs.io/en/stable/public_server.html](http://jupyter-notebook.readthedocs.io/en/stable/public_server.html)

Secure Apache: [https://geekflare.com/10-best-practices-to-secure-and-harden-your-apache-web-server/](https://geekflare.com/10-best-practices-to-secure-and-harden-your-apache-web-server/)


## Notes
1. I built this guide on Ubuntu 16.04.1 LTS. However, the process should be very similar for users with another version or other Linux distros. Just follow the guidelines for installing the various software on your system. For Mac users the process should also be kind of similar. Windows users should use this guide as a rough guide and swing a bit around with the different ways to install and setup stuff.

2. I assume you run Ubuntu 16.04.
