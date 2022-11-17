Title: Using Caddyserver on Debian11 to reverse proxy a legacy HTTPS site: no need for Certbot
Author: Ferdinando Simonetti
Tags: Debian11, Caddyserver, Reverse Proxy
Date: 2022-11-17

# Caddyserver setup on Debian 11

## Credits/bibliography
I've been following Caddyserver's official site guidance here

https://caddyserver.com/

### Installing Caddy

```
ferdi@fsmntest0:~$ sudo apt update
ferdi@fsmntest0:~$ sudo apt upgrade -y
ferdi@fsmntest0:~$ sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
ferdi@fsmntest0:~$ curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
ferdi@fsmntest0:~$ curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
# Source: Caddy
# Site: https://github.com/caddyserver/caddy
# Repository: Caddy / stable
# Description: Fast, multi-platform web server with automatic HTTPS


deb [signed-by=/usr/share/keyrings/caddy-stable-archive-keyring.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main

deb-src [signed-by=/usr/share/keyrings/caddy-stable-archive-keyring.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main

ferdi@fsmntest0:~$ sudo apt update
ferdi@fsmntest0:~$ sudo apt install caddy
```
### Testing Caddy reverse proxying from command line

You have to stop the running daemon first; after that, it's a very intuitive command line.
```
ferdi@fsmntest0:~ $ sudo systemctl stop caddy
ferdi@fsmntest0:~ $ sudo caddy reverse-proxy \
>  --from newsite.fsmn.xyz \
>  --to https://blog.fsmn.xyz \
>  --change-host-header
```
Et voilà! HTTPS certificate requested to Let's Encrypt by Caddy on the fly! 

### How to convert this command in a Caddyfile suitable to systemd daemon?

To be continued...
