# nextcloud with OnlyOffice in Docker
# Only Office
For quite some time there has been a built-in community server of OnlyOffice, OO, for processing office documents. It doesn't work properly and is also a bit sluggish. Therefore I have created some workaraounds, which will be discussed at the end. To make things stop, I installed an independent server via *Docker*, where I also encountered some problems, which will be mentioned in the following chapter.

During the research and through comments from commentators two different ways to install the Docker server were approached:
- Docker Server, which is directly accessible from the net and therefore a port in the router has to be released. However, no further LE certificate needs to be created here.
- Docker Server, which can be accessed via a reverse proxy through NGINX. No further port needs to be released for this, however, another LE certificate must be created.

**Both variants are equally secure if the additional port in the router will be ignored.** So it remains the personal preference, which variant is chosen.
## Docker Server - directly accessible
It was not quite satisfactory and also the community version was a bit behind the current version and so first [Docker installed](https://docs.docker.com/engine/install/debian/ "Official Docker documentation with installation under Debian") and then the [Docker server installed](https://helpcenter.onlyoffice.com/server/docker/document/docker-installation.aspx "Official documentation of OnlyOffice with the Docker installation").

But this was the beginning of the problems:

1. I wanted to make the server accessible via HTTPS. The official instructions provide self-signed certificates, which are blocked by the browsers in the meantime. Here a solution using Let's Encrypt was necessary.
2. for HTTPS another port had to be used.
3. the OnlyOffice server should be additionally secured with a key.

In the internet there was a tutorial for the setup with LE, but it did not really work. Here is the command for setting up the docker container:
>sudo docker pull onlyoffice/documentserver

>sudo docker run -i -t -d --name="uniquername" -p local_port:443 \--restart=always \-v /app/onlyoffice/DocumentServer/logs: /var/log/onlyoffice \-v /app/onlyoffice/DocumentServer/data:/var/www/onlyoffice/Data \-v /app/onlyoffice/DocumentServer/lib:/var/lib/onlyoffice \-e JWT_ENABLED="true" \-e JWT_SECRET="own_secret" onlyoffice/documentserver

**Important:**

- *name* must be written in "".
- A free local port must be used. This must also be enabled in the firewall (e.g. 'ufw') and the router.
- *BEFORE* each new argument must be a `\` and then NO space
- The arguments for the two 'JWDs' must be in "".
- The 'JWT_SECRET' should not include any quotation marks e.g. '"'. Otherwise, Docker interprets this as the end of the string and then Nextcloud will break up with the passphrase.

Especially the last two points are very important, otherwise the container will not run correctly and the cloud will deliver an error message. Googling this error message does NOT help.

Afterwards you have to create the directories for the `certs` and everything not existing yet. For the directory for the cerst you have to make sure that the rights are correct.

Furthermore the existing certificates of LE must be copied into the directory of the container:
>
	cd /app/onlyoffice/DocumentServer/data/certs/
	cp -L /etc/letsencrypt/live/a.domain/cert.pem onlyoffice.crt
	cp -L /etc/letsencrypt/live/a.domain/privkey.pem onlyoffice.key

and the rights adjusted '444' and the correct owner and group are changed according to the parent directories!! The latter is especially important, because otherwise the following thing will not work and the page view will not work. The chain must still be added:
>
	wget https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem.txt
	cat lets-encrypt-x3-cross-signed.pem.txt>>onlyoffice.crt

The 'cat' command is particularly sensitive in terms of rights. So if this does not work, just check the rights of the `onlyoffice.crt`. If everything works, the test, if the server is running by calling the port *local_port* should work with the domain of the certificate. In addition, it should be checked whether the 'index.html' can be called up, as this is important for the communication between the cloud and the OO server:
> wget https://eine.domain:lokaler_Port

If this fails, the setup in the cloud will also fail. Read the error message from 'wget' and do it. Often it is due to the missing chain and 'cat' because the LE certificate is for a different domain.

## Docker Server - Reverse Proxy
Here too, the reasons and problems were the same in this version, whereby 2. could be omitted, since HTTPS is via the same port. However,documents could not be loaded as quickly with this version as with the other version. It is only milliseconds, but it was noticeable.

Here is the recommendation for setting up the Docker Container:
>sudo docker pull onlyoffice/documentserver

>sudo docker run -i -t -d --name="unambiguous name" -p 127.0.0.1:local_port:80 --restart=always -e JWT_ENABLED="true" -e JWT_SECRET="secret_secret" onlyoffice/documentserver

**Important:**

- *name* must be written in "".
- A free local port must be used.This must also be enabled in the firewall (e.g. 'ufw') and it must be written before the ':80'.
- The arguments for the two 'JWT' must be in "".
- The 'JWT_SECRET' should not include any quotation marks e.g. '"'. Otherwise, Docker interprets this as the end of the string and then Nextcloud will break up with the passphrase.

Especially the last point is very important, otherwise the container will not run correctly and the cloud will deliver an error message. Googling this error message does NOT help. You get many hits, but no real help.

If you want to run several OO-servers for different clouds, just create a new container and assign a new name and port. I have not yet tested a server for several clouds.

### Setting up the reverse proxy with NGINX
In order for the OO server to be accessible, the reverse proxy must be set up in NGINX. There are also [instructions for Apache](https://arnowelzel.de/en/onlyoffice-in-nextcloud-the-current-status "Thank you for the link!"), but I haven't tested them because I use NGINX.

The setup consists of three steps:
1. config for accessibility with port 80
2. get LE certificate with Certbot
3. rewrite Conf for the reverse proxy

For normal accessibility with port 80 just write a normal Conf for NGINX. There you can take everything that is there at the moment:
>
	server {
		listen 80;
		listen [::]:80;

		server_name a.unique.domain;

		access_log /var/log/nginx/your-access_log;
		error_log /var/log/nginx/your-error_log crit;

		root /var/www/html;
		index index.html index.htm index.nginx-debian.html
	}

The Certbot can then be run. If '---nginx' is given as an option, the corresponding changes are taken over.

Finally adjust the Conf to the reverse proxy. You have to add a 'location' section with the proxy pass information. These are a little more complex if the actual domain is accessible via HTTPS. I have listed the complete Conf here to show you the changes Certbot has made:
>
	server {

		server_name a.unique.domain;
		
		access_log /var/log/nginx/your-access_log;
		error_log /var/log/nginx/your-error_log crit;

		location / {
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header Host $http_host;
			proxy_set_header X-Forwarded-Proto https;
			proxy_redirect off;
			proxy_pass http://127.0.0.1:lokaler_Port;
			proxy_http_version 1.1;
		}

		listen [:::]:443 ssl http2; # managed by Certbot
		listen 443 ssl; # managed by Certbot
		ssl_certificate /etc/letsencrypt/live/a.unique.domain/fullchain.pem; # managed by Certbot
		ssl_certificate_key /etc/letsencrypt/live/a.unique.domain/privkey.pem; # managed by Certbot
		include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
		ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
	}
>
	server {
	# if ($host = a.unique.domain) {
	# return 301 https://$host$request_uri;
	# } # managed by Certbot

		listen 80;
		listen [::]:80;

		server_name a.unique.domain;
		return 404; # managed by Certbot

		return 301 https://$server_name$request_uri;
	}

**Important:**

- With 'proxy_pass' you have to enter the same address including port as the docker was used before.
- In the official documentation of NGINX it is mentioned that the `if` rule for redirect is not recommended. It is better to use the `return 301`. Therefore it was commented out and added below.

## Setting up the cloud
Afterwards the cloud has to be set up. In the OnlyOffice app the fields are filled in as follows, which is the same for both variants:

- service address: Address of the cloud with the port of the container behind it -> *https://domain.der.cloud:lokaler_Port* or *https://eine.eindeutige.domain*
- Secret key: The secret secret from the above 'JWT_SECRET'.
- Under "Advanced server settings": Service address of the document processing for internal requests from the server: The same as before for the service address
- Under "Advanced server settings": Server address for internal requests from the document processing service: The address of the cloud as it can be reached normally.

Then click on "Save" and the matter should be clear without changes.

In the internet some more things were found, which should help to solve problems. These were no longer necessary after the correct docker configuration. But I would like to list them here for the sake of completeness:
In the 'config.php' in the cloud the following things can be added individually or in combination:
>
	'onlyoffice' => 
	array(
		'jwt_secret' => 'secret_secret',
		'jwt_header' => 'Authorization',
		'verify_peer_off' => TRUE,
	),

Especially 'verify_peer_off' is mentioned very often.

If `jwt_secret` is used, this is automatically written into the field *secret key* in the setting. Everything else was not obvious to me

### Renewing the certificates within the container
The certificates of LE expire after 90 days. Therefore the files must be exchanged within the container. A script is sufficient for this:
>
	#!/bin/bash
	cp /etc/letsencrypt/live/a.domain/fullchain.pem /app/onlyoffice/DocumentServer/data/certs/onlyoffice.crt
	cp /etc/letsencrypt/live/a.domain/privkey.pem /app/onlyoffice/DocumentServer/data/certs/onlyoffice.key
	chmod 400 /app/onlyoffice/DocumentServer/data/certs/onlyoffice.key
	docker container restart docker_name

This then only needs to be processed via Cron:
> 2 */12 * * * root /bin/bash --login /path/to/script.sh > /dev/null 2>
