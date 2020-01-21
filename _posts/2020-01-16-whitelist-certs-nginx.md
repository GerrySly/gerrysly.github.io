---
layout: post
title: Certificate Whitelisting in Nginx
category: DevOps
---

Nginx has to be one of my all time favourite pieces of software. I remember reading about it and it's architecture in the book, "The Architecture of Open Source (Volume 2)", and being thoroughly impressed with their hot restarting mechanism. Another fantastic article on the structuring of nginx is, "[Inside NGINX: How We Designed for Performance & Scale](https://www.nginx.com/blog/inside-nginx-how-we-designed-for-performance-scale/)", I highly encourage you to read one or the other!

One of the things I think makes nginx super powerful is it's concise yet highly customizable ways in which you can configure it to behave. This leads it to not only working as a web server but a reverse proxy, load balancer, mail proxy, cache and if you extend it using it's plugin system the potential is limitless. This article however will delve into some of the TLS capabilities of nginx and more specifically using it as a mTLS access point.

## ```ngx_http_ssl_module```

The TLS module in Nginx has 24 directives to allow you to perform all sorts of TLS offloading and does it incredibly well! A typical setup for TLS in your ```nginx.conf``` might look something like this

```
server {
  listen 443 ssl http2;
  ...

  # Path to certs
  ssl_protocols           TLSv1.2 TLSv1.3;
  ssl_certificate         /etc/nginx/ssl/certificate.pem;
  ssl_certificate_key     /etc/nginx/ssl/private_key.key;
  ssl_client_certificate  /etc/nginx/ssl/trusted_ca.chain;
  ssl_verify_client       on
  ...
}
```

Now given I am just going to be talking about certificate whitelisting, I am omitting some core directives required for this to get working but essentially this configuration is doing the following actions:

```ssl_protocols TLSv1.3;```
Telling nginx that we are only going to accept browsers capable of handshaking using TLSv1.2 and TLSv1.3 standard, this allows you to tell nginx to avoid any standards which may be out of date or have any vulnerabilities exposed.

```ssl_certificate /etc/nginx/ssl/certificate.pem;```
Providing a location for the certificate that we wish to present when someone goes to our server. This must be in PEM format.

```ssl_certificate_key /etc/nginx/ssl/private_key.key;```
The private key which nginx will use to setup the TLS secured connection to the client (Please see [What Happens in a TLS handshake](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/)).

```ssl_client_certificate /etc/nginx/ssl/trusted_ca.chain;```
This is simply a CA that you want to trust for your clients. This is where, for my specific use case, I supply our internal root CA which is not provided in any regular CA store but for all internal services that present a certificate, must be signed by this CA.

```ssl_verify_client on```
Simply verifices that the clients certificate was signed by the ```ssl_client_certificate``` list of CA's

## Whitelisting

So although there is not much to setting up TLS on an nginx server, one of the awesome things that it exposes comes in the [embedded variables](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#variables). These variables are allowed to be used in the server context block to perform further logic on the established TLS connection. The final ... block in the example ```nginx.conf``` file above would typically contain some routes. Something that might look like this

```
location / {
  proxy_pass http://127.0.0.1:8080;
}
```

Simply put, all routes that being with / are proxied straight through to localhost on port 8080. Now with our configuration listed above and the help of our ```ssl_client_certificate``` directive we can reject any consumers who do not present a certificate signed by our internal CA. This, however, does not help because given the size of our organisation, we cannot be certain that our list of consumers are all going to be non-malicious actors.

This leads us to the embedded variable ```$ssl_client_s_dn``` if you want to go a level deeper you can actually get back the ENTIRE certificate in the embedded variable ```$ssl_client_escaped_cert```

Basically from here we can use the if conditional to check if the supplied subject DN of the consumer matches any from our list of allowed consumers. Simply put:

```
location / {
  if ($ssl_client_s_dn_cn !~ "CN=www.example.com,OU=orgUnit,O=org,L=Melbourne,ST=VIC,C=AU") {
    return 401;
  }

  proxy_pass http://127.0.0.1:8080;
}
```

That is simply it. You can now copy paste that if condition multiple times to match multiple subject DN's of multiple consumers. In fact, one of the automated ways in which we've performed this is simply having a shell script which takes an allowed subject DN, validates it's format and then injects a precrafted if condition into nginx in runtime.
