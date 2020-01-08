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

The process by which I achieve mTLS with multiple certificates can really be more easily rationalised as mTLS with multiple DN's.
