# Getting an SSL Certificate
Getting an SSL certificate can range from super easy to surprisingly more complicated than expected. If an AWS instance is already using the Elastic load balancer (ELB), it seems like it&#39;s possible to get a SSL certificate through the [AWS Certificate Manager](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/ssl-server-cert.html). But for small applications that can&#39;t justify spending on the ELB, or if someone wants to set up their own load balancer, things can become much more involved.

----
### Options for Node.js applications
While there are a number of ways to go about getting an SSL certificate for a Node.js application, it can largely be binned into two categories: Either have the application itself or a reverse proxy (e.g. Nginx) handle the certificate. When it comes to enabling SSL/TLS, it seems that using an Nginx reverse proxy is the recommend approach [by Express](https://expressjs.com/en/advanced/best-practice-security.html). Moreover, placing Nginx in front of a Node application has a number of benefits, as [outlined here](https://www.nginx.com/blog/5-performance-tips-for-node-js-applications/). This includes enabling load balancing, mitigating DDoS attacks, caching static pages and other functions that could improve performance.

----
### Prepping for a reverse proxy
In order to go the reverse proxy route, both a Nginx and a Certbot container will be required, in addition to the application itself. A general overview for launching multi-container applications can be found [here](https://github.com/atcsrepo/Misc-notes/blob/master/EC2-Multi-Container-Launch.md). This write-up will only focus on the SSL certificate aspect, and will be an amalgamation of a number of online guides, such as [this](https://www.digitalocean.com/community/tutorials/how-to-secure-a-containerized-node-js-application-with-nginx-let-s-encrypt-and-docker-compose#step-4-%E2%80%94-obtaining-ssl-certificates-and-credentials) and [this](https://medium.com/&#64;pentacent/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71).

#### Certbot Validation and Nginx
Let&#39;s Encrypt uses the ACME protocol[&#x00B9;](https://tools.ietf.org/html/draft-ietf-acme-acme-07#page-48),[&#x00B2;](https://community.letsencrypt.org/t/whats-the-file-that-http-01-challenge-requires-at-challenge/42342/2) to validate that a web server can control a given domain. What happens is that Certbot creates an ACME account + keypair with Let&#39;s Encrypt and requests for validation. Certbot will receive a token from Let&#39;s Encrypt which it uses to generate a key authorization string. The string is stored within a file and placed at `/.well-known/acme-challenge/<TOKEN>`. Let&#39;s Encrypt will download the file and verify its contents. If the key authorization is valid, then certificates will be authorized for the given domain. 

As such, a Nginx container and a Certbot container will need to share a resource path (e.g. `/var/www/html`) so that Certbot can place a file that  Nginx can point to. Likewise, both will need to share the location at which the keys/certificates will be stored (`/etc/letsencrypt`) and also the Certbot working directory (`/var/lib/letsencrypt`). Creating a persistent volume for access logs (`/var/log/letsencrypt`) is optional.

----
### Setting Nginx up for the challenge

Nginx will fail to start if SSL is enabled and no certificates are present. There are two options to get around this. First, just configure the `nginx.conf` file so that Nginx only listens on port 80 to begin with. In other words, leave out everything that involves HTTPS/port 443. After Certbot acquires the staging or production certificates, update `nginx.conf` to include re-directing to https and reload the proxy. At this point, Nginx should run normally.

Alternatively, it is also possible to generate [self-signed certificates](https://stackoverflow.com/questions/10175812/how-to-create-a-self-signed-certificate-with-openssl "self-signed certificates") for set-up purposes, then remove the invalid certificates and restart Nginx after Certbot generates valid certificates.

---
### Automating renewal and restarts
Certificates from Let&#39;s Encrypt have a 90-day validity period. While it is certainly possible to manually update it every 3 months and restart the Nginx instance, it does become a chore. As we are using Docker containers, there are two options - either set up a cron job (or some equivalent) to run the container or start up the container with an automation script.

----
### A super quick example
The following is a simplified example derived from an application running on an AWS EC2 instance. This is meant to serve as a quick outline of the steps involved.

Initial pseudo-YAML/docker-compose file:

```yaml
version: "3"

services: 
    app:
        image: <repo link>
        command: ["pm2-runtime", "server.js"]
    revproxy:
        image: <repo link>
        ports:
            - "80:80"
            - "443:443"
        volumes:
            - /var/www/html:/var/www/html
            - /etc/letsencrypt:/etc/letsencrypt
            - /var/lib/letsencrypt:/var/lib/letsencrypt
        links: 
            - app:app
        command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
    certbot:
        image: certbot/certbot
        volumes:
            - /var/www/html:/var/www/html
            - /etc/letsencrypt:/etc/letsencrypt
            - /var/lib/letsencrypt:/var/lib/letsencrypt
        entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```
Note the start up scripts that are used for Nginx and Cerbot. Certbot will wake up every 12h to check certicates while Nginx will re-load itself every 6 hrs. After setting up the YAML file, check the `nginx.conf` file and edit the server block to allow for listening on port 80 only; likewise, disable/remove anything HTTPS-related. For example:

```
server {
        listen 80;
        server_name  _;
        
        location ~ /.well-known/acme-challenge {
            allow all;
            root /var/www/html;
        }
        
        location / {
            proxy_pass http://app:8000;
        }
    }
```
Start up the containers with `docker-compose` or some equivalent and confirm that everything is running properly using `ecs ps` or `docker ps`. 
In order to test Certbot and the Nginx, we will need to SSH into the EC2 instance and run Certbot like so:

```
docker run -v  /var/www/html:/var/www/html -v /etc/letsencrypt:/etc/letsencrypt -v /var/lib/letsencrypt:/var/lib/letsencrypt certbot/certbot certonly --webroot --webroot-path=/var/www/html --register-unsafely-without-email --agree-tos --no-eff-email --staging -d example.com  -d www.example.com
```
If registering with an e-mail, use `--email someone@example.com` instead of `--register-unsafely-without-email`. Likewise, add or remove additional domains as needed. If everthing is configured properly, there should be a congratulatory message, like the following:
```
certbot_1  |  - Congratulations! Your certificate and chain have been saved at:
certbot_1  |    /etc/letsencrypt/live/example.com/fullchain.pem
certbot_1  |    Your key file has been saved at:
certbot_1  |    /etc/letsencrypt/live/example.com/privkey.pem
certbot_1  |    Your cert will expire on 2019-05-04. To obtain a new or tweaked
certbot_1  |    version of this certificate in the future, simply run certbot
certbot_1  |    again. To non-interactively renew *all* of your certificates, run
certbot_1  |    "certbot renew"
```
At this point, update the nginx.conf file to include HTTPS re-directing:
```
server {
        listen 80;
        server_name  _;
        
        location ~ /.well-known/acme-challenge {
            allow all;
            root /var/www/html;
        }
        
        location / {
            return 301 https://$host$request_uri;
        }
    }
	
server {
	listen 443 ssl http2;
	server_name example www.example.com;
	
	ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
	
	location / {
		proxy_pass http://app:8000;
	}
}
```
and restart Nginx.

It should now be possible to access the website/application via a public HTTP address and be re-directed to an HTTPS address with an invalid SSL certificate (assuming that staging certificates are being used).

Once everything checks out, use:
```
docker run -v  /var/www/html:/var/www/html -v /etc/letsencrypt:/etc/letsencrypt -v /var/lib/letsencrypt:/var/lib/letsencrypt certbot/certbot certonly --webroot --webroot-path=/var/www/html --register-unsafely-without-email --agree-tos --no-eff-email --force-renewal -d example.com  -d www.example.com
```
within the EC2 instance to replace the staging certificates with production certificates. 

Restart Nginx again and HTTP requests should now be re-directed to an HTTPS address with a valid certificate. The SSL renewal and reloading process should take place automatically with the provided entrypoint and command scripts provided in the YAML file, so there&#39;s no need to schedule a task.
