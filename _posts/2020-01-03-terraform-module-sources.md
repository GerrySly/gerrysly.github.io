---
layout: post
title: Terraform Module Sources
category: DevOps
---

This past week and a bit I've been really digging into [Terraform module sources](https://www.terraform.io/docs/modules/sources.html) and their ability to externalise the module that you rely upon.

It's been pretty powerful for my workflow as the team I'm currently working in are moving towards a more microservices oriented way of working, there tends to be a lot of reuse of infrastructure. For example, the first module that I built out was an AWS module which simply stands up an Network Load Balancer and deploys an ECS task onto an already existing ECS cluster. I simply plug this module into all my container based workloads and no more reinventing the wheel and any learnings these individual microservices gain along their journey can be shared amongst all common infrastructures.

## Enterprise GitHub Module Source

One of the painful tasks I did come across was when I attempted to get the module sources working in a CI/CD environment (Jenkins specifically). Part of my workflow usually involves running all my commands in a docker container to allow a common environment to be leveraged and not relying on all the things to be available in my Jenkins slaves.

So you can see where I'm going here, I wanted to run terraform in a container. So I simply pulled down a terraform container with the latest version, mounted my infra file (with a module and the source being our AWS hosted Enterprise GitHub) and much to my chagrin my ```terraform init``` command failed miserably.

Essentially within the terraform container it has a version of ```git``` that it relies on that (obviously now) doesn't share the same credentials nor keys as the git that is used to setup the workspace in Jenkins.

### ASK_GITPASS

The Jenkins environment has currently two configured ways of accessing the Enterprise GitHub
1. SSH keys setup to allow ```git``` access via SSH
2. A preconfigured GitHub OAuth token to allow the Jenkins GitHub plugin to function

I decided to leverage the GitHub OAuth token and a https module source along with the Jenkins [Credentials Binding Plugin](https://jenkins.io/doc/pipeline/steps/credentials-binding/) to obfuscate the token in the pipeline. Once I had the GitHub OAuth token, I needed to get the terraform ```git``` to use the token. There was [quite a few different ways of achieving this](https://coolaj86.com/articles/vanilla-devops-git-credentials-cheatsheet/). I decided to go with the GIT_ASKPASS environment variable approach. I encourage you to have a look at the previous page, it is chock full of helpful information about the subject. These two choices led me to the following code snipped.

```groovy
environment {
  GIT_ASKPASS = "${WORKSPACE}/.git-askpass"
}

withCredentials(
  [
    usernamePassword(
      credentialsId: 'github-token',
      usernameVariable: 'GITHUB_USERNAME',
      passwordVariable: 'GITHUB_TOKEN'
    )
  ]
) {
  sh """
    echo 'echo "${GITHUB_TOKEN}"' > "${WORKSPACE}/.git-askpass
    chmod +x "${WORKSPACE}/.git-askpass

    terraform init -auto-approve -input=false
  """
}
```

What is cool about this is that the GIT_ASKPASS environment variable is invoked and the result is passed into git when the https Enterprise GitHub is used. Just to show you an idea of what the source looked like in Terraform module

```terraform
module "my-arbitrary-module" {
  source = "git::https://enterprise.github.example.com/ORGANISATION/terraform-modules.git?ref=MY_BRANCH"
}
```

When I run the ```terraform init``` comand it invokes a ```git checkout``` and because a https URL is supplied it will ask for a password. Now because ASK_GITPASS is setup, it will invoke this file and pass the result from that command into the password field. 
