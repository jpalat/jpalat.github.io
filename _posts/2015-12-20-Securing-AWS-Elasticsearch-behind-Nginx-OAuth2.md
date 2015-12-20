---
layout: post
title: Securing AWS Elasticsearch behind Nginx + OAuth2
---
Securing AWS Elasticsearch behind Nginx + OAuth2.
=================================================

We are currently using AWS as our Elastic Search Provider.  Rather than allowing this service to be open from anywhere on the internet, we have used security groups to protect it and allow it to only accept traffic from our ETL box.  We are also using the AWS instance of kibana.  To support this, we added nginx as a reverse-proxy.  This improves things, but still allows anyone on the internet to access our server.  To foil would-be attackers we're going to protect our server using Oauth2.  This also reduces drag on our internal users by reducing the need for another new username and password.

I wanted to include a SSL certificate and decided to try the new Let's Encrypt Library.

So how is this going to work?  It requires 5 components


1. AWS Elasticsearch
2. Nginx
3. Let's Encrypt
4. An Oauth2 provider
5. oauth2_proxy 


Elasticsearch
-------------
For the purposes of this article I will assume you already have an Elasticsearch instance.  If you are not using the AWS Elasticsearch you may need to add your own ports on top of these services.  In this article I will use `https://ELASTICSEARCH_CLOUD.amazonaws.com/` to refer to my ES instance.  Note that it uses the https port(443) and not the standard ES ports.  If you are using your own instance you may need to add the appropriate ports.

Nginx
-----

We are running on a stock Ubuntu 14.04 stack.  Our nginx installation is also stock so to install simply:
``sudo apt-get install nginx``

SSL with Let's Encrypt
----------------------

[Let's Encrypt](https://letsencrypt.org/) is a free, automated, and open certificate authority (CA), run for the public’s benefit. Let’s Encrypt is a service provided by the Internet Security Research Group (ISRG).  At this time Let's Encrypt does not support nginx automatically, but the basic pieces for putting it together are pretty straightforward.

1) Download letsencrypt.

    $ git clone https://github.com/letsencrypt/letsencrypt
    $ cd letsencrypt


2) Setup letsencrypt.

Let's encrypt uses python virtualenvs to take care of the heavy lifting of cert creation.  The first step is to let it bootstrap itself.

``./letsencrypt-auto run``

3) To generate certificate for nginx we need to generate a standalone set of certificates:

``./letsencrypt-auto certonly --standalone``


4) Verify that the certifcates have been created

``sudo ls /etc/letsencrypt/live/``
``sudo ls /etc/letsencrypt/live/mydomain.com``


Setting up SSL with nginx
-------------------------

The next step is to set up nginx as a reverse proxy.  We create a nginx config in /etc/ngnix/sites-available/proxy 

1) First we set the server to listen for SSL traffic
with the following contents:


    server {
      listen                443 ssl;
      server_name  mydomain.com;
    
      ssl_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/mydomain.com/privkey.pem;
    
      ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
      ssl_prefer_server_ciphers on;
      ssl_ciphers AES256+EECDH:AES256+EDH:!aNULL;
    }


2) Next we setup the proxy.  Within the server stanza add a location stanza similar to this one

      location /{
        proxy_set_header        Host $host;
        proxy_set_header        X-Real-IP $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;
        proxy_redirect http:// https://;
        proxy_pass https://ELASTICSEARCH_CLOUD.amazonaws.com/; # elastic search node
      }
 

As it stands you can now connect to elasticsearch instance using a cloud provider.  The proxy matches any traffic for '/' and forwards it to our cloud server. It will also redirect http traffic to https.  

Setting up OAuth2 Provider
--------------------------

1. Sign in to https://console.developers.google.com
2. Create a new Project
3. In the new Project, select "Enable and Manage APIs"
4. Go to the "Credentials" menu
5. New Credential > OAuth Client ID
6. Configure the Consent Screen
	- Product name shown to users

This will take you back to the screen to create Credentials

Select "Web Application"
Name: ExampleApp

Authorized JavaScript origins:
https://www.example.com

Authorized redirect URIs:
https://www.example.com/oauth2/callback

It is *critical* to provide a Authorized redirect URI.

Google will then provide a clientID and client secret.  Keep these close, we'll need them soon.




Oauth2 Proxy
------------

Oauth2 Proxy provides all the heavy lifting for dealing with OAuth.
To install Download [Prebuilt Binary](https://github.com/bitly/oauth2_proxy/releases) (current release at the time of this writing is v2.0.1) 
Copy oauth2_proxy to /usr/bin

The architecture I worked with is:

     ---------      ----------     -----------------
     | ngnix | ____ | Oauth2 | ___ | Elasticsearch |
     |  SSL  |      | Proxy  |     |     Kibana    |
     ---------      ----------     -----------------
                        |
                        |
                    -----------
                    | google  |
                    | Oauth   |
                    -----------

Setting up the proxy
--------------------
To setup the proxy I used this 

	    oauth2_proxy  -client-id=<CLIENTID> \
	              -client-secret=<CLIENT SECRET> \
	              -email-domain=mydomain.com \
	              -upstream=https://ELASTICSEARCH_CLOUD.amazonaws.com/ \
	              -cookie-secret=secretsecret \
	              -provider="google"\
	              -cookie-domain=<SERVER_NAME> \
	              -cookie-secure=true \
	              -pass-basic-auth=false \
	              -pass-access-token=false


The header for basic auth that oauth_proxy provides is not correctly interpreted by Elasticsearch.  I turned -pass-basic-auth to false to allow my requests to be properly parsed by Elasticsearch.  Once the proxy is up and running we need to update nginx to point to the new proxy instead of our Elasticsearch endpoint.  The proxy is now listening locally at port 4180.

in /etc/ngnix/sites-available/proxy we're going to update our proxy pass from
``    proxy_pass https://ELASTICSEARCH_CLOUD.amazonaws.com/; # elastic search node``

to 

``    proxy_pass http://127.0.0.1:4180; # oauth2_proxy node``

we restart nginx and we should now be prompted to authenticate with Google.


Sources
-------
* https://letsencrypt.org/howitworks/
* https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04
* https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-with-ssl-as-a-reverse-proxy-for-jenkins

* https://wiki.jenkins-ci.org/display/JENKINS/Jenkins+behind+an+NGinX+reverse+proxy

* https://github.com/bitly/oauth2_proxy#google-auth-provider
* http://canalplus.github.io/blog/2015/11/07/install-nginx-reverse-proxy-with-github-oauth2/

