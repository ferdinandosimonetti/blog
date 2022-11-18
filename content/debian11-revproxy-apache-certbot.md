Title: Using Debian 11's Apache to reverse proxy a legacy HTTPS site, with SSL certificate provided by Certbot/Let'sEncrypt
Author: Ferdinando Simonetti
Tags: Debian11, Apache, Reverse Proxy, Certbot
Category: UNIX
Date: 2022-11-16
Modified: 2022-11-18

# Apache reverse proxy setup on Debian 11

## Credits/bibliography
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
        ServerName newsite.fsmn.xyz
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
        ServerName newsite.fsmn.xyz
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
### Requesting SSL certificate to Let's Encrypt
Just like that! The certificate will be requested and fetched (and a cron job will be created to take care of renewals) and our **revproxy.conf** will be automagically modified by:
- adding **RewriteEngine/RewriteCond/RewriteRule** to redirect HTTP calls to HTTPS
- referencing the new certificate
```
ferdi@testdebian1:~$ sudo certbot --apache -d newsite.fsmn.xyz
...
```
### Reloading Apache and test
To have our new configuration in place, you have to reload Apache
```
sudo systemctl reload apache2
```
Now you can test the redirection via browser, pointing at **https://newsite.fsmn.xyz**
When you're satisfied with your tests, it's time to 

### Add oldsite.fsmn.xyz as ServerAlias inside your revproxy.conf
Just below the ServerName directive on both VirtualHosts (**\*:80** and **\*:443**), add the line
```
ServerAlias oldsite.fsmn.xyz
```
And reload Apache
```
sudo systemctl reload apache2
```
Now it's time to switch **oldsite.fsmn.xyz** DNS entry to point at the new IP **(better doing that off-hours)**.
### Expanding Certbot-provided SSL certificate
Next (quick) action should be to update our SSL certificate to include oldsite.fsmn.xyz.
```
ferdi@testdebian1:~$ sudo certbot --apache -d newsite.fsmn.xyz -d oldsite.fsmn.xyz
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
You have an existing certificate that contains a portion of the domains you
requested (ref: /etc/letsencrypt/renewal/www.fsmn.xyz.conf)

It contains these names: newsite.fsmn.xyz

You requested these names for the new certificate: newsite.fsmn.xyz, oldsite.fsmn.xyz.

Do you want to expand and replace this existing certificate with the new
certificate?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(E)xpand/(C)ancel: e
Renewing an existing certificate for newsite.fsmn.xyz and 1 more domain

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/newsite.fsmn.xyz/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/newsite.fsmn.xyz/privkey.pem
This certificate expires on 2023-02-15.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for newsite.fsmn.xyz to /etc/apache2/sites-enabled/revproxy.conf
Successfully deployed certificate for oldsite.fsmn.xyz to /etc/apache2/sites-enabled/revproxy.conf
Your existing certificate has been successfully renewed, and the new certificate has been installed.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```
and reloading Apache as usual (just in case)
```
sudo systemctl reload apache2
```
