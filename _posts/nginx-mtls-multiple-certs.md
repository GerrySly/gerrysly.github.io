---
layout: post
title: mTLS with Nginx and Multiple Certificates
---

Nginx is a fantastic piece of software that can be used for a wide range of applications. For the use case that I've been deploying it as are mainly centered around TLS termination and mTLS verification to a downstream client (in this case a Kong gateway which, funnily enough, is [also based on nginx](https://konghq.com/faqs/)). 

Just to set the scene of where Nginx has been deployed in our architecture, I've included the following diagram to illustrate as much:

![mTLS Diagram](/public/mTLS_diagram.png)

For the purposes of this post, the Kong API Gateway internals are of little importance and the actual infrastructure underpinning the solution are also of little importance. What I will highlight now is, the mechanism by which the Nginx sidecar and the Express container communicate is via the docker network, this level of isolation means we do not expose the Express port and only allow those within that network to see the port. The only exposed port via this is the Nginx port.

## Limitations

Now before I delve into my ```nginx.conf``` I wanted to preface it with some limitations this solution has that we deem as accepted risks within our own architecture. 

1. All the certificates that are presented MUST be signed by the same Certificate Authority (CA). For our purposes, the mTLS is being implemented to lock down to either clients presenting as Kong OR our own intra-VPC clients that comprise our service mesh. If you're wishing to perform mTLS with multiple certificates signed by different CA's, this post is not for you.

2. The checks that I perform are only at a CA and Distinguished Name (DN) level. I do not complete any thumbprint validation and the validation on a DN level is purely string matching.

## Process

The process by which I achieve mTLS with multiple certificates can really be more easily rationalised as mTLS with multiple DN's. The basic building block that we leverage within nginx is the [ngx_http_ssl_module](http://nginx.org/en/docs/http/ngx_http_ssl_module.html) and it's associated directives. The process that is completed occurs as such:

1. First when the ServerHello message is sent by nginx back to the client (in our example the client is our Kong gateway) it includes the public certificate. This is the first step in the mutual part of mTLS.

2. Our client (Kong in this case) verifies that the certificate is one signed by our internal Root CA and that it is also coming from a whitelisted DN that it supports. If at this point the client is not happy with the identity of the server (e.g. it presents a self-signed certificate) it can reject the identity and not continue with the TLS handshake

3. At the same time that the nginx (server) responds with it's own certificate, it send a packet requesting the clients certificate, the client can concurrently respond back with it's own certificate, if it does not nginx can reject the client at this point with 403 Forbidden.

4. This is where the magic happens. Nginx will not validate the CA of the presented certificate (leveraging the ```ssl_trusted_certificate``` directive in nginx). Assuming the CA was valid, there is now a variable set which represents the clients DN, ```$ssl_client_s_dn```. We can now use a basic if condition to validate if the DN presented matches one we know. If it does not match we return a 403

## Multiple DNs

Nginx configuration files have limited capacity for complex conditional logic and as such you have to get a bit creative with how you lay out your config file. For our particular setup, we leverage entrypoint scripts in our Dockerfile's such that they bootstrap the ```nginx.conf``` file. One such example is the usage of an environment variable called ```$MTLS_ENABLED``` that will perform some shell-fu to enable or disable mTLS. The following is a snippet of our bootstrap script that performs the logic

```shell
if [ -z "${MTLS_ENABLED:-}" ]; then
  if [[ "${MTLS_ENABLED}"]]
  sed -i '' 's/__MTLS__ENABLED__/ENABLED_/g' /etc/nginx/nginx.conf
else
  sed -i '' 's/__MTLS__ENABLED__/DISABLED/g' /etc/nginx/nginx.conf
fi
```

Basically what this is doing is in-place replacing any occurrence of the token ```__MTLS__ENABLED__``` within the ```nginx.conf``` with the ENABLED_ token.
