
# XWiki 16.10 with Jetty 12.0.15 Installation Guide for Ubuntu Server 24.04
Installation xwiki knowledge base on ubuntu server with postgres DB


## official installation manual:

https://www.xwiki.org/xwiki/bin/view/Documentation/AdminGuide/Installation/InstallationWAR/InstallationJetty/

## Prerequisites

- Ubuntu Server (24.04 LTS or newer)
- Java 17 or Java 21 (preferably OpenJDK)
- Postgres DB

## Step-by-Step Installation

### 1\. Update System and Install Java

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-21-jdk -y
# or older version
# sudo apt install openjdk-17-jdk -y
```

### 2\. Verify Java Installation

```bash
java -version
```

### 3\. Create Installation Directory

```bash
mkdir -p /opt/xwiki
cd /opt/xwiki
```

### 4\. Download Jetty

```bash
wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/12.0.15/jetty-home-12.0.15.tar.gz

```

### 4.1 Unzip Jetty

```bash
tar -xzvf jetty-home-12.0.15.tar.gz
```

### 5\. Download XWiki

```bash
wget https://github.com/xwiki/xwiki-platform/releases/download/xwiki-16.10/xwiki-platform-distribution-war-16.10.war
```

### 6\. Prepare Jetty Base Directory

```bash
mkdir -p /opt/xwiki/jetty-base
cd /opt/xwiki/jetty-base
```

### 7\. Create Jetty Base Configuration

```bash
# pwd = /opt/xwiki/jetty-base
java -jar /opt/xwiki/jetty-home-12.0.15/start.jar --create-startd
```

### 8\. Enable Necessary Jetty Modules

List of required modules:

```
ext
resources
server
logging
http
http-forwarded
ee8-annotations
ee8-deploy
requestlog
ee8-websocket-javax
ee8-websocket-jetty
ee8-apache-jsp
console-capture
```

```bash
java -jar /opt/xwiki/jetty-home-12.0.15/start.jar --module=server
java -jar /opt/xwiki/jetty-home-12.0.15/start.jar --module=deploy
java -jar /opt/xwiki/jetty-home-12.0.15/start.jar --module=ext,resources,server,console-capture,http,http-forwarded,ee8-annotations,ee8-deploy,ee8-webapp,requestlog,ee8-websocket-javax,ee8-websocket-jetty,ee8-apache-jsp,console-capture
```

### 9\. Prepare XWiki Deployment

```bash
mkdir -p /opt/xwiki/jetty-base/webapps/
unzip -q /opt/xwiki/xwiki-platform-distribution-war-16.10.war -d /opt/xwiki/jetty-base/webapps/
```

### 10\. optionally we can configure Jetty Startup (цей файл я не створював)

Create `/opt/xwiki/jetty-base/start.ini` with the following content:

```
# Jetty Base Configuration
--module=server
--module=deploy
--module=ee8-apache-jsp
--module=ee8-websocket-javax
--module=ee8-websocket-jetty

# XWiki Specific Configuration
-Djetty.home=/opt/xwiki/jetty-home-12.0.15
-Djetty.base=/opt/xwiki/jetty-base

# JVM Options
-Xmx2g
-Xms512m
```

### 11\. Set Permissions

```bash
sudo chown -R $(whoami):$(whoami) /opt/xwiki
```

### 12\. Install and configure Postgres database

12.1. Install Postgres

```
sudo apt install postgresql
```

12.2. Connect to Postgres DB

```
sudo su postgres
psql
```

12.3. Create postgres database "xwiki" and user account "xwiki"

```
create user xwiki password 'XXXXXXXXX';
create database xwiki with owner=xwiki;
grant all on schema public to xwiki;
\q
```

12.4 Check connection to the DB

```
psql postgresql://xwiki:'password'@localhost
```

12.5 Copy postgres java driver to xwiki project folder

```
wget https://jdbc.postgresql.org/download/postgresql-42.7.3.jar
# copy to the folder /opt/xwiki/jetty-base/webapps/xwiki/WEB-INF/lib
```

12.6 Configure xwiki application to work with postgres DB (edit file xwiki/WEB-INF/hibernate.cfg.xml)

```bash
cd /opt/xwiki/jetty-base/webapps/xwiki/WEB-INF/
vim hibernate.cfg.xml
```

Comment default database, uncomment and configure postgres:

```
    <property name="hibernate.connection.url">jdbc:postgresql://localhost:5432/xwiki</property>
    <property name="hibernate.connection.username">xwiki</property>
    <property name="hibernate.connection.password">XXXXXXXXXXX</property>
    <property name="hibernate.connection.driver_class">org.postgresql.Driver</property>
    <property name="hibernate.jdbc.use_streams_for_binary">false</property>
    <property name="xwiki.virtual_mode">schema</property>

    <property name="hibernate.connection.charSet">UTF-8</property>
    <property name="hibernate.connection.useUnicode">true</property>
    <property name="hibernate.connection.characterEncoding">utf8</property>

    <mapping resource="xwiki.postgresql.hbm.xml"/>
    <mapping resource="feeds.hbm.xml"/>
    <mapping resource="instance.hbm.xml"/>
    <mapping resource="notification-filter-preferences.hbm.xml"/>
    <mapping resource="mailsender.hbm.xml"/>
```

### 13.  Create permanent directory for XWIKI

```bash
mkdir /var/lib/xwiki/data
```

Add user permissions

```bash
sudo useradd xwiki

sudo chown -R $(whoami):$(whoami) /var/lib/xwiki
# sudo chown xwiki:xwiki /var/lib/xwiki

```

### 14\. Edit xwiki settings for permanent directory

```bash
/opt/xwiki/jetty-base/webapps/xwiki/WEB-INF/xwiki.properties
```

set variable:  
`environment.permanentDirectory = /var/lib/xwiki/data/`

### 15\. Edit jetty file /opt/xwiki/jetty-base/start.d/server.ini

find string  jetty.httpConfig.uriCompliance=default

set:  
`jetty.httpConfig.uriCompliance=RFC3986`

### 16\. Systemd Service configuring

Create `/etc/systemd/system/xwiki.service`:

```ini
[Unit]
Description=XWiki Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/xwiki/jetty-base
ExecStart=/usr/bin/java -jar /opt/xwiki/jetty-home-12.0.15/start.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 13\. Enable and Start XWiki Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable xwiki
sudo systemctl start xwiki
```

### 14\. Configure Firewall (if needed)

```bash
sudo ufw allow 8080/tcp
```

## HGINX setup for secure https connection

https://www.rosehosting.com/blog/how-to-install-xwiki-on-ubuntu-22-04/#Step-5-Install-SSLTLS-Certificate

Install nginx service

```bash
apt install nginx
```

In Ubuntu, nginx will automatically start and is enabled on boot. Now, let’s create a server block for our Xwiki website.

```bash
vim /etc/nginx/sites-enabled/xwiki.conf
```

Then, paste the following into the file.

```bash
map $request_uri $expires {
default off;
~*\.(ogg|ogv|svg|svgz|eot|otf|woff|mp4|ttf|rss|atom|js|jpg|jpeg|gif|png|ico|zip|tgz|gz|rar|bz2|doc|xls|exe|ppt|tar|mid|midi|wav|bmp|rtf)(\?|$) 1h;
~*\.(css) 0m;
}

expires $expires;

server {
   listen 80;
   server_name yourdomain.com;
   charset utf-8;

   root /var/www/html;

   location /.well-known {
     alias /var/www/html;
   }

   location / {
     rewrite ^ $scheme://$server_name/xwiki$request_uri? permanent;
   }

   location ^~ /xwiki {
     proxy_pass http://localhost:8080;
     proxy_cache off;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header Host $http_host;
     proxy_set_header X-Forwarded-Proto $scheme;
     expires $expires;
   }
}
```

Make sure to replace yourdomain.com with your actual domain or subdomain name before saving the file and then exiting. Next, let’s check the configuration and restart nginx.

```
nginx -t
systemctl restart nginx
```

## Install SSL/TLS Certificate

In this step, we are going to install a free SSL certificate from Let’s Encrypt. Let’s install certbot to continue with this.

```
apt install certbot python3-certbot-nginx -y
```

Now it is time to execute this command below to generate an SSL/TLS certificate from Let’s Encrypt.

```
certbot
```

You will get an output like this; you need to answer the prompts.

```
[root@ubuntu22]# certbot

Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
(Enter 'c' to cancel): you@yourdomain.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.3-September-21-2022.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: n
Account registered.

Which names would you like to activate HTTPS for?
We recommend selecting either all domains, or all domains in a VirtualHost/server block.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: yourdomain.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel): 1
Requesting a certificate for yourdomain.com

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/yourdomain.com/fullchain.pem
Key is saved at: /etc/letsencrypt/live/yourdomain.com/privkey.pem
This certificate expires on 2023-03-04.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for yourdomain.com to /etc/nginx/conf.d/yourdomain.com.conf
Congratulations! You have successfully enabled HTTPS on https://yourdomain.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
* Donating to ISRG / Let's Encrypt: https://letsencrypt.org/donate
* Donating to EFF: https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

&nbsp;

## Accessing XWiki

Open a web browser and navigate to:  
`http://your-server-ip:8080`

## Troubleshooting Tips

- Check logs at `/opt/xwiki/jetty-base/logs/`
- Verify Java paths and versions
- Ensure all dependencies are correctly installed

## Post-Installation

- Complete initial setup through web interface
- Configure database connection
- Set up admin account

* * *

**Note:** Adjust paths and versions as needed for your specific environment.

&nbsp;
