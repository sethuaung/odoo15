Odoo is a popular open-source suite of business apps that help companies to manage and run their business. It includes a wide range of applications such as CRM, e-Commerce, website builder, billing, accounting, manufacturing, warehouse, project management, inventory, and much more, all seamlessly integrated.

Odoo can be installed in different ways, depending on the use case and available technologies. The easiest and quickest way to install Odoo is by using the official Odoo APT repositories.

Installing Odoo in a virtual environment, or deploying as a Docker container, gives you more control over the application and allows you to run multiple Odoo instances on the same system.

This article goes through installing and deploying Odoo 15 inside a Python virtual environment on Ubuntu. We’ll download Odoo from the official GitHub repository and use Nginx as a reverse proxy.

# Installing Dependencies

Create a Python 3.9 Virtual Environment Ubuntu – Learn IT And DevOps Daily

The first step is to install Git , Pip , Node.js , and development [tools required to  build]

(https://itslinuxfoss.com/how-to-install-gcc-on-ubuntu-20-04/ )

## Odoo dependencies:

```
$sudo apt update
```

```
$sudo apt install git python3-pip build-essential wget python3-dev python3-venv \

python3-wheel libfreetype6-dev libxml2-dev libzip-dev libldap2-dev libsasl2-dev \

python3-setuptools node-less libjpeg-dev zlib1g-dev libpq-dev \

libxslt1-dev libldap2-dev libtiff5-dev libjpeg8-dev libopenjp2-7-dev \

liblcms2-dev libwebp-dev libharfbuzz-dev libfribidi-dev libxcb1-dev
```
## Creating a System User

Running Odoo under the root user poses a great security risk. We’ll create a new system user and group with home directory /opt/odoo15 that will run the Odoo service. To do so, run the following command:
```
$sudo useradd -m -d /opt/odoo15 -U -r -s /bin/bash odoo15
```
You can name the user anything you want, as long you create a PostgreSQL user with the same name.

# Installing and Configuring PostgreSQL

Odoo uses PostgreSQL as the database back-end. PostgreSQL is included in the standard Ubuntu repositories. The installation is straightforward:

```
$sudo apt install postgresql
```
Once the service is installed, create a PostgreSQL user with the same name as the previously created system user. In this example, that is odoo15:

````
$sudo su - postgres -c "createuser -s odoo15"
````

**_Installing wkhtmltopdf_**

wkhtmltopdf is a set of open-source command-line tools for rendering HTML pages into PDF and various image formats. To print PDF reports in Odoo, you’ll need to install the wkhtmltox package.

The version of wkhtmltopdf that is included in Ubuntu repositories does not support headers and footers. The recommended version for Odoo is version 0.12.5. We’ll download and install the package from Github:
```
$sudo wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.bionic_amd64.deb
````
Once the file is downloaded, install it by typing:
```
$sudo apt install ./wkhtmltox_0.12.5-1.bionic_amd64.deb
````

## Installing and Configuring Odoo 15

We’ll install Odoo from the source inside an isolated Python virtual environment .

First, change to user “odoo15”:
```
$sudo su - odoo15
````
Clone the Odoo 15 source code from GitHub:

```
$git clone https://www.github.com/odoo/odoo --depth 1 --branch 15.0 /opt/odoo15/odoo
````
Create a new Python virtual environment for Odoo:

```
$cd /opt/odoo15
$python3 -m venv odoo-venv
```
Activate the virtual environment:
```
$source odoo-venv/bin/activate
```
Odoo dependencies are specified in the requirements.txt file. Install all required Python modules with pip3:
```
(odoo-venv) $pip3 install wheel
(odoo-venv) $pip3 install -r odoo/requirements.txt
```

If you encounter any compilation error during the installation, make sure all required dependencies listed in the Installing Prerequisites section are installed.

Once done, deactivate the environment by typing:
```
(odoo-venv) $deactivate
```
We’ll create a new directory a separate directory for the 3rd party addons:
```
$mkdir /opt/odoo15/odoo-custom-addons
```
Later we’ll add this directory to the addons_path parameter. This parameter defines a list of directories where Odoo searches for modules.

Switch back to your sudo user:
```
$exit
```
Create a configuration file with the following content:
```
$sudo nano /etc/odoo15.conf
```
```
[options]

; This is the password that allows database operations:

admin_passwd = my_admin_passwd

db_host = False

db_port = False

db_user = odoo15

db_password = False

addons_path = /opt/odoo15/odoo/addons,/opt/odoo15/odoo-custom-addons
```

Do not forget to change the my_admin_passwd to something more secure.

## Creating Systemd Unit File

A unit file is a configuration ini-style file that holds information about a service.

Open your text editor and create a file named odoo15.service with the following content:
```
$sudo nano /etc/systemd/system/odoo15.service
```
```
[Unit]

Description=Odoo15

Requires=postgresql.service

After=network.target postgresql.service

[Service]

Type=simple

SyslogIdentifier=odoo15

PermissionsStartOnly=true

User=odoo15

Group=odoo15

ExecStart=/opt/odoo15/odoo-venv/bin/python3 /opt/odoo15/odoo/odoo-bin -c /etc/odoo15.conf

StandardOutput=journal+console

[Install]

WantedBy=multi-user.target
```

Notify systemd that a new unit file exists:

```
$sudo systemctl daemon-reload
```
Start the Odoo service and enable it to start on boot by running:
```
$sudo systemctl enable --now odoo15
```
Verify that the service is up and running:
```
$sudo systemctl status odoo15
```
The output should look something like below, showing that the Odoo service is active and running:
```
● odoo15.service - Odoo15

Loaded: loaded (/etc/systemd/system/odoo15.service; enabled; vendor preset: enabled)

Active: active (running) since Tue 2023-10-26 09:56:28 UTC; 28s ago

...
```
You can check the messages logged by the Odoo service using the command below:
```
$sudo journalctl -u odoo15
```

## Testing the Installation

Open your browser and type: _http://<your_domain_or_IP_address>:8069_

Assuming the installation is successful, a screen similar to the following will appear:



### Configuring Nginx as SSL Termination Proxy

The default Odoo web server is serving traffic over HTTP. To make the Odoo deployment more secure, we will set Nginx as an SSL termination proxy that will serve the traffic over HTTPS.

SSL termination proxy is a proxy server that handles the SSL encryption/decryption. This means that the termination proxy (Nginx) will process and decrypt incoming TLS connections (HTTPS), and pass on the unencrypted requests to the internal service (Odoo). The traffic between Nginx and Odoo will not be encrypted (HTTP).

Using a reverse proxy gives you a lot of benefits such as Load Balancing, SSL Termination, Caching, Compression, Serving Static Content, and more.

Ensure that you have met the following prerequisites before continuing with this section:

Domain name pointing to your public server IP. We’ll use example.com.

Nginx installed .

SSL certificate for your domain. You can install a free Let’s Encrypt SSL certificate .

Open your text editor and create/edit the domain server block:
```
$sudo nano /etc/nginx/sites-enabled/felixent.com.conf
```
The following configuration sets up SSL Termination, HTTP to HTTPS redirection , WWW to non-WWW redirection, cache the static files, and enable GZip compression.

/etc/nginx/sites-enabled/felixent.com.conf
```
# Odoo servers

upstream odoo {

server 127.0.0.1:8069;

}

upstream odoochat {

server 127.0.0.1:8072;

}

# HTTP -> HTTPS

server {

listen 80;

server_name www.felixent.com felixent.com;

include snippets/letsencrypt.conf;

return 301 https://felixent.com$request_uri;

}

# WWW -> NON WWW

server {

listen 443 ssl http2;

server_name www.felixent.com;

ssl_certificate /etc/letsencrypt/live/felixent .com/fullchain.pem;

ssl_certificate_key /etc/letsencrypt/live/felixent .com/privkey.pem;

ssl_trusted_certificate /etc/letsencrypt/live/felixent .com/chain.pem;

include snippets/ssl.conf;

include snippets/letsencrypt.conf;

return 301 https://felixent .com$request_uri;

}

server {

listen 443 ssl http2;

server_name felixent .com;

proxy_read_timeout 720s;

proxy_connect_timeout 720s;

proxy_send_timeout 720s;

# Proxy headers

proxy_set_header X-Forwarded-Host $host;

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

proxy_set_header X-Forwarded-Proto $scheme;

proxy_set_header X-Real-IP $remote_addr;

# SSL parameters

ssl_certificate /etc/letsencrypt/live/felixent .com/fullchain.pem;

ssl_certificate_key /etc/letsencrypt/live/felixent .com/privkey.pem;

ssl_trusted_certificate /etc/letsencrypt/live/felixent .com/chain.pem;

include snippets/ssl.conf;

include snippets/letsencrypt.conf;

# log files

access_log /var/log/nginx/odoo.access.log;

error_log /var/log/nginx/odoo.error.log;

# Handle longpoll requests

location /longpolling {

proxy_pass http://odoochat;

}

# Handle / requests

location / {

proxy_redirect off;

proxy_pass http://odoo;

}

# Cache static files

location ~* /web/static/ {

proxy_cache_valid 200 90m;

proxy_buffering on;

expires 864000;

proxy_pass http://odoo;

}

# Gzip

gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;

gzip on;

}
```
Don’t forget to replace example.com with your Odoo domain and set the correct path to the SSL certificate files. The snippets used in this configuration are created in this guide .

Once you’re done, restart the Nginx service :
```
$sudo systemctl restart nginx
```
Next, we need to tell Odoo to use the proxy. To do so, open the configuration file and add the following line:

/etc/odoo15.conf
```
proxy_mode = True
```
Restart the Odoo service for the changes to take effect:
```
$sudo systemctl restart odoo15
```
At this point, the reverse proxy is configured, and you can access your Odoo instance at https://felixent .com.

### Changing the Binding Interface

This step is optional, but it is a good security practice.

By default, the Odoo server listens to port 8069 on all interfaces. To disable direct access to the Odoo instance, you can either block port 8069 for all public interfaces or force Odoo to listen only on the local interface.

We’ll configure Odoo to listen only on 127.0.0.1. Open the configuration add the following two lines at the end of the file:

/etc/odoo15.conf
```
xmlrpc_interface = 127.0.0.1

netrpc_interface = 127.0.0.1
```
Save the configuration file and restart the Odoo server for the changes to take effect:
```
$sudo systemctl restart odoo15
```
## Enabling Multiprocessing

By default, Odoo is working in multithreading mode. For production deployments, it is recommended to change to the multiprocessing server as it increases stability and makes better usage of the system resources.

To enable multiprocessing, you need to edit the Odoo configuration and set a non-zero number of worker processes. The number of workers is calculated based on the number of CPU cores in the system and the available RAM memory.

According to the official Odoo documentation , to calculate the workers’ number and required RAM memory size, you can use the following formulas and assumptions:

**Worker number calculation**

Theoretical maximal number of worker = (system_cpus * 2) + 1
```
1 worker can serve ~= 6 concurrent users

Cron workers also require CPU

RAM memory size calculation
```
We will consider that 20% of all requests are heavy requests, and 80% are lighter ones. Heavy requests are using around 1 GB of RAM while the lighter ones are using around 150 MB of RAM

_Needed RAM = number_of_workers * ( (light_worker_ratio * light_worker_ram_estimation) + (heavy_worker_ratio * heavy_worker_ram_estimation) )_

If you do not know how many CPUs you have on your system, use the following grep command:
```
$grep -c ^processor /proc/cpuinfo
```
Let’s say you have a system with 4 CPU cores, 8 GB of RAM memory, and 30 concurrent Odoo users.
```
30 users / 6 = **5** (5 is theoretical number of workers needed )

(4 * 2) + 1 = **9** ( 9 is the theoretical maximum number of workers)
```
Based on the calculation above, you can use 5 workers + 1 worker for the cron worker that is a total of 6 workers.

Calculate the RAM memory consumption based on the number of workers:
```
RAM = 6 * ((0.8*150) + (0.2*1024)) ~= 2 GB of RAM
```
The calculation shows that the Odoo installation will need around 2GB of RAM.

To switch to multiprocessing mode, open the configuration file and append the calculated values:

/etc/odoo15.conf
```
limit_memory_hard = 2684354560

limit_memory_soft = 2147483648

limit_request = 8192

limit_time_cpu = 600

limit_time_real = 1200

max_cron_threads = 1

workers = 5
```

Restart the Odoo service for the changes to take effect:
```
$sudo systemctl restart odoo15
```
The rest of the system resources will be used by other services that run on this system. In this guide, we installed Odoo along with PostgreSQL and Nginx on the same server.
