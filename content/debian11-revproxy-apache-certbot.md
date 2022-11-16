Title: Using Debian 11's Apache to reverse proxy a legacy HTTPS site, with SSL certificate provided by Certbot/Let'sEncrypt
Author: Ferdinando Simonetti
Tags: Debian11, Apache, Reverse Proxy, Certbot
Date: 2022-11-16

# Apache setup on Debian 11

I've been following this site's hints here
https://blog.containerize.com/2021/05/21/how-to-configure-apache-as-a-reverse-proxy-for-ubuntudebian/

## Assumptions
- Your old, not updateable combo of webserver and application has IP address **192.168.105.37**
- It's currently published on the Internet and its public DNS' FQDN is **theoldsite.fsmn.xyz**
- Your new server can reach the Internet and is reachable from there on both 80/TCP and 443/TCP
- You can manage public DNS records

### Apache installation and activation of reverse proxy and SSL modules
```
sudo apt update -y
sudo apt upgrade -y
sudo apt install -y apache2
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod lbmethod_byrequests
sudo a2enmod ssl
sudo systemctl restart apache2
```
### Create initial site config, HTTP->HTTP
You'll have to create a new *\*.conf* file under */etc/apache2/sites-available/*
```
# revproxy.conf
<VirtualHost *:80>
        ServerName newname.fsmn.xyz
        ServerAdmin webmaster@fsmn.xyz
        ErrorLog ${APACHE_LOG_DIR}/rp-error.log
        CustomLog ${APACHE_LOG_DIR}/rp-access.log combined
        ProxyRequests Off
        <Proxy *>
          Order deny,allow
          Allow from all
        </Proxy>
        ProxyPass / http://theoldsite.fsmn.xyz/
        ProxyPassReverse / http://theoldsite.fsmn.xyz/
        <Location />
          Order allow,deny
          Allow from all
        </Location>
</VirtualHost>

<IfModule mod_ssl.c>
<VirtualHost *:443>
        ServerName newname.fsmn.xyz
        ServerAdmin webmaster@fsmn.xyz
        ErrorLog ${APACHE_LOG_DIR}/rp-error.log
        CustomLog ${APACHE_LOG_DIR}/rp-access.log combined

        ProxyRequests Off
        SSLProxyEngine On
        <Proxy *>
          Order deny,allow
          Allow from all
        </Proxy>
        ProxyPass / https://theoldsite.fsmn.xyz/
        ProxyPassReverse / https://theoldsite.fsmn.xyz/

        <Location />
          Order allow,deny
          Allow from all
        </Location>
</VirtualHost>
</IfModule>
```
and add a line to your */etc/hosts* so that **your reverse proxy host** can resolve **the old site** (possibly via an **internal** IP address)
```
192.168.105.37 theoldsite.fsmn.xyz
```
### Enabling the reverse proxy Apache configuration, disable the default Apache 
Now you should enable the new 'site' and disable the default one
```
ferdi@testdebian1:~$ sudo a2dissite 000-default
Site 000-default disabled.
To activate the new configuration, you need to run:
  systemctl reload apache2
ferdi@testdebian1:~$ sudo a2ensite revproxy
Enabling site revproxy.
To activate the new configuration, you need to run:
  systemctl reload apache2
ferdi@testdebian1:~$ sudo systemctl reload apache2
```
### Adding temporary DNS entry to test the HTTPS setup
You have to make sure that **newsite.fsmn.xyz** resolves to your **new server**'s public IP
### to be continued... :-)