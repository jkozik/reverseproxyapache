# Apache Reverse Proxy with Lets Encrypt SSL setup with certbot -webroot

For my home network, I have only one IP address and therefore one port 80/443 for all of my home web services.  This repository documents how I setup an Apache reverse proxy to route port 80/443 traffic to the various different services I support.  This includes a blogs, storage, media and websites.  Some running on standalone servers, others running in a kubernetes cluster.

The reverse proxy needs to do the following steps:
- Define a virtual host per application and redirect it to the internal server and port number
- Redirect www to non-www (www.example.com -> example.com)
- Map http requests to https
- Use certbot to get and renew certificates from [Let's Encrypt](https://letsencrypt.org/)

I have an existing Apache httpd server -- I am adding the reverse proxy function to it as a series of proxy.appname.conf files. The way reverse proxys work, I cannot redirect an https requests.  I must terminate the SSL at the reverse proxy and forward the traffic to it on port 80.

# Apache-based Reverse Proxy Virtual Hosts
THe reverse proxy is listening on port 80 and 443, getting traffic from my cable-modem/home router.  Apache httpd takes the incoming packets and maps them to virtual hosts, in this case, based on host name.  The [ProxyPass directive ](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html#proxypass) invokes a reverse proxy function and redirects the traffic. Here's a very simple example for my weather web page http://napervilleweather.com.

```
<VirtualHost *:80>
 ServerName napervilleweather.com
 ServerAlias www.napervilleweather.com
 ProxyPreserveHost On
 ProxyPass / http://192.168.100.174:30140/
 ProxyPassReverse / http://192.168.100.174:30140/
</VirtualHost>
```
In this particular case, the ProxyPass/ProxyPassReverse directives are forwarding the traffic to my kubernetes cluster.  

I need to make a virtual host configuration for each of my many services.  I am just showing one example.

# Redirect www to non-www
I have decided that I want to force all services to redirect to a URL without the www prefix.  That is www.napervilleweather.com -> napervilleweather.com.
Apache httpd [rewrite engine](https://httpd.apache.org/docs/2.4/rewrite/intro.html) can do this. [See example.](https://www.digitalocean.com/community/tutorials/how-to-redirect-www-to-non-www-with-apache-on-centos-7).  Here's how the virtual host configuration file changes
```
...
 RewriteEngine on
 RewriteCond %{SERVER_NAME} =www.napervilleweather.com
 ReWriteRule ^ http://napervilleweather.com%{REQUEST_URI} [END,QSA,R=permanent]
 ...
 ```
 I am pretty sure the rewrite rules get run before the ProxyPass ones. 
 
 # Redirect http to https
 
 I am also redirecting http to https. In summary, 
 - http://napervilleweather.com -> https://napervilleweather.com
 - http://www.napervilleweather.com -> https://napervilleweather.com
 - https://www.napervilleweather.com -> https://napervilleweather.com

 Thus the apache httpd configuration file changes
 ```
 ...
 RewriteEngine on
 RewriteCond %{SERVER_NAME} =www.napervilleweather.com [OR]
 RewriteCond %{SERVER_NAME} =napervilleweather.com
 ReWriteRule ^ https://napervilleweather.com%{REQUEST_URI} [END,QSA,R=permanent]
 ...
 ```
 
 
 From the redirects above, the apache virtual server napervilleweather.com:443 terminates the SSL and redirects the web traffic to my kubernetes cluster on my home LAN http://192.168.100.174:30140/.  This final redirection is done by the ProxyPass/ProxyPassReserve commands and completes the reverse proxy function.
 
# certificates from Lets Encrypt using certbot client
I use the cerbot client to create the certificates for napervilleweather.com.  To do that, I choose to use the webroot plugin.  This lets me create and renew the certificate by using a path within the website to support the handshake with the Let's Encrypt server:  napervilleweather.com/.well-known/acme-challenge. The certbot tool temporarily stores some credentials there and the server gets them to verify that the person running the certbot tool has control of the website.  

Here's an example certbot execution:
```
[root@dell1 sites-enabled]# certbot certonly --cert-name napervilleweather.com --webroot -w /var/lib/letsencrypt/http_ch
allenges/napervilleweather.com -d napervilleweather.com -d www.napervilleweather.com --dry-run
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator webroot, Installer None
Starting new HTTPS connection (1): acme-staging-v02.api.letsencrypt.org
Cert not due for renewal, but simulating renewal for dry run
Simulating renewal of an existing certificate for napervilleweather.com and www.napervilleweather.com
Performing the following challenges:
http-01 challenge for napervilleweather.com
http-01 challenge for www.napervilleweather.com
Using the webroot path /var/lib/letsencrypt/http_challenges/napervilleweather.com for all unmatched domains.
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - The dry run was successful.
[root@dell1 sites-enabled]#
```

Note: the certificate for napervilleweather.com also includes www.napervilleweather.com.  Also note the above command was run as "dry-run".

For this to work I setup a webroot path to an existing letsencrypt directory on the reverse proxy server.  The real web server is inside my kubernetes cluster.  To make this work, I need to expand the virtual server definition to map .well-known/acme-challenge to a local directory and not let the reverse proxy forward the traffic.  See the additional configuration file directives:
```
 DocumentRoot /var/lib/letsencrypt/http_challenges/napervilleweather.com
  <Directory /var/lib/letsencrypt/http_challenges/napervilleweather.com>
          Allow from All
  </Directory>
  <Location /.well-known/acme-challenge>
      Require all granted
      Options Indexes
  </Location>
 ...
 ProxyPreserveHost On
 ProxyPass /.well-known !
 ProxyPass / http://192.168.100.174:30140/
 ProxyPassReverse / http://192.168.100.174:30140/
 ...
```

The certbot -w /var/lib/letsencrypt/http_challenges/napervilleweather.com command line option tells certbot where the put the temporary creditials. The DocumentRoot directive tells the apache httpd where the root of the webserver is stored and where one can find the .well-known/acme-challenge directory.


# References
- [How To Redirect www to Non-www with Apache on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-redirect-www-to-non-www-with-apache-on-centos-7)
- [Www giving error SSL_ERROR_BAD_CERT_DOMAIN](https://community.letsencrypt.org/t/www-giving-error-ssl-error-bad-cert-domain/81073)
- [[SOLVED]NET::ERR_CERT_COMMON_NAME_INVALID on www](https://community.letsencrypt.org/t/solved-net-err-cert-common-name-invalid-on-www/96629)
- https://check-your-website.server-daten.de/?q=napervilleweather.com
- [Setup free SSL certificate for MiaRec using Let's Encrypt (Centos 6/7)](https://www.miarec.com/book/export/html/926)
- [certbot - get help](https://certbot.eff.org/pages/help#webserver)
- [Welcome to the Certbot documentation!](https://eff-certbot.readthedocs.io/en/stable/)
- [Apache ProxyPass directive](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html#proxypass)
- [cerbot -- Problem binding to port 80 with –standalone](https://community.letsencrypt.org/t/problem-binding-to-port-80-with-standalone/50850)
- [Certbot not creating acme-challenge folder](https://stackoverflow.com/questions/38382739/certbot-not-creating-acme-challenge-folder)
- [Error message "Forbidden You don't have permission to access / on this server"](https://stackoverflow.com/questions/10873295/error-message-forbidden-you-dont-have-permission-to-access-on-this-server)
- [Apache 2.4: mod_alias, mod_rewrite, mod_proxy execution order](https://serverfault.com/questions/966675/apache-2-4-mod-alias-mod-rewrite-mod-proxy-execution-order)
- [Redirect HTTP to HTTPS in Apache](https://linuxize.com/post/redirect-http-to-https-in-apache/)
- 
