---
layout: post
title: File Sharing between Docker Containers
category: DevOps
---

So this post will be short and sweet but I just wanted to document something awesome I came across yesterday regarding Docker and how to share files between two containers which are running.

## Problem

Imagine you have a 2 pipelines, one is to build your artefact and the other is to deploy the build artefact. The former runs in every commit webhook and has all the bells and whistles you could imagine around unit testing, integration testing, static code analysis, security scanning, linting etc. etc. and once successful pushes a certified artefact into an artefact repository. The latter pipeline has all the infrastructure related activities that simply takes that certified artefact and deploys it onto your cloud provider.

The particular problem I was having was relating to the deployment to the API gateway. Essentially in our first pipeline (build) we validate and verify that our API definitions (defined by [OpenAPI 3.0](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md) specifications) are well formed and meet all the standards and criteria set out by our API Gateway team. The team provides some tooling for this purpose and we run it in our build pipeline. This means that the artefact we push has our correct and well-formed OpenAPI API definitions baked into the containers.

Now when we go to run the deploy pipeline, we run into a problem. The CI/CD tool that I am using is Jenkins and as such when a pipeline is ran it sources a Jenkinsfile from git and checks out the repository. We now have 2 versions of the OpenAPI 3.0 API definitions, one we just checked out which could potentially be out of date to the definitions that have been baked into the container. 

## Docker ```--volumes-from``` Flag

This leads me to the solution, one of the awesome things that you can do within Docker is mount volumes from another container into a running container using the ```--volumes-from``` flag. 

So basically to solve my problem, I simply ran my container built in the build pipeline and setup a Docker volume pointing to the same directory that contains my OpenAPI 3.0 API Definitions. 

```bash
docker run --rm --detach \
  --volume /app/api-definitions \
  --name openapi-defs \
    BUILD_CONTAINER:tag
```

This becomes the first step in my pipeline so that I now have a Docker volume I can leverage in my API Gateway tooling container to pull in the API definitions whilst maintaining the integrity of all the goodness we performed in our build pipeline.

This means for our API tooling container to utilise the API definitions itself, we can simply use the following

```bash
docker run --rm \
  --volumes-from openapi-defs \
  --workdir /app/api-definitions \
  API_TOOLING_CONTAINER:tag \
    ...to run...
```

Now within the tooling container at the directory ```/app/api-definitions``` we have our validated and well-formed API definitions coming from our build artefact at the time of build. 

I like to run a post step after this to clean up the container just in case any errors pop up, and I use a Jenkins ```post``` section for this

```groovy
post { 
    always { 
        sh """
          docker kill openapi-defs || true
        """
    }
}
```
