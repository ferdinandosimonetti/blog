Title: Using Caddyserver on Debian11 to reverse proxy a legacy HTTPS site: no need for Certbot
Author: Ferdinando Simonetti
Tags: Debian11, Caddyserver, Reverse Proxy
Category: UNIX
Date: 2022-11-17
Modified: 2022-11-18

Using a more modern and lightweight way for dealing with obsolete combos of OSs and application versions.

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
2022/11/17 09:01:52.330 WARN    admin   admin endpoint disabled
2022/11/17 09:01:52.330 INFO    http    server is listening only on the HTTPS port but has no TLS connection policies; adding one to enable TLS {"server_name": "proxy", "https_port": 443}
2022/11/17 09:01:52.330 INFO    http    enabling automatic HTTP->HTTPS redirects        {"server_name": "proxy"}
2022/11/17 09:01:52.330 INFO    tls.cache.maintenance   started background certificate maintenance      {"cache": "0xc000688bd0"}
2022/11/17 09:01:52.330 INFO    http    enabling HTTP/3 listener        {"addr": ":443"}
2022/11/17 09:01:52.330 INFO    tls     cleaning storage unit   {"description": "FileStorage:/root/.local/share/caddy"}
2022/11/17 09:01:52.330 INFO    failed to sufficiently increase receive buffer size (was: 208 kiB, wanted: 2048 kiB, got: 416 kiB). See https://github.com/lucas-clemente/quic-go/wiki/UDP-Receive-Buffer-Size for details.
2022/11/17 09:01:52.330 INFO    http.log        server running  {"name": "proxy", "protocols": ["h1", "h2", "h3"]}
2022/11/17 09:01:52.330 INFO    http.log        server running  {"name": "remaining_auto_https_redirects", "protocols": ["h1", "h2", "h3"]}
2022/11/17 09:01:52.330 INFO    http    enabling automatic TLS certificate management   {"domains": ["newsite.fsmn.xyz"]}
2022/11/17 09:01:52.331 INFO    tls     finished cleaning storage units
Caddy proxying https://newsite.fsmn.xyz -> blog.fsmn.xyz:443
```
Et voilà! HTTPS certificate requested to Let's Encrypt by Caddy on the fly!
But there's an INFO message suggesting some system/kernel tuning, namely this one
```
ferdi@fsmntest0:/etc/caddy$ sudo sysctl -w net.core.rmem_max=2500000
net.core.rmem_max = 2500000
``` 
(of course you'll have to set it permanently).
Subsequent **caddy** run with same parameters doesn't show this hint anymore.

### How to convert this command in a Caddyfile suitable to systemd daemon?

Thanks to Markus Dosch for his blog article here

https://www.markusdosch.com/2022/05/caddy-web-server-why-use-it-how-to-use-it/

I've set up the main **/etc/caddy/Caddyfile** and the directory structure reminiscent of Apache's own **/etc/caddy/sites-available**.

The main **Caddyfile** contains now global options like my own email address, and an *import* directive.

```
root@fsmntest0:/etc/caddy/sites-enabled# cat /etc/caddy/Caddyfile
{
    email info@fsimonetti.info
}

import /etc/caddy/sites-enabled/*.caddy
```

My **revproxy.caddy** (inside *sites-enabled* subdir) is here:

```
newsite.fsimonetti.info {
    log {
        output file /var/log/caddy/blog-access.log
    }
    reverse_proxy {
        to https://blog.fsimonetti.info
        header_up Host {upstream_hostport}
    }
}
```

The most important directive here is the **header_up** that, according to official documentation found here 

https://caddyserver.com/docs/caddyfile/directives/reverse_proxy#https

Enables us to rewrite the **Host** header of the request going to the upstream (backend for other reverse proxies out there) with **its own FQDN** as declared in **to** directive above.