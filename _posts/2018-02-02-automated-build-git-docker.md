---
layout: post
title:  "CI / CD using github, Docker and Kubernetes"
date:   2018-02-02 12:00:00
categories: [Docker]
tags: [docker, netcoreapp, cloud, k8s]
draft: false
---
In my [last post]({% post_url 2018-01-24-docker-aspnet-core-3 %}) on this topic I got my basic ASP.NET Core Docker container up and running locally. The next step is to get it deployed to the Cloud! To do this in the quickest way possible I decided to use [Docker Cloud](https://docs.docker.com/docker-cloud/) for my automated build and to store my Docker image, and [Azure Container Service (AKS)](https://azure.microsoft.com/en-gb/services/container-service/) to host the application.

## Automated build
The first step is to [create a repository](https://docs.docker.com/docker-cloud/builds/repos/#create-a-new-repository-in-docker-cloud) in Docker Cloud to host my Docker image. You can just use this as a registry and push any existing Docker images to it, but I decided to [set up an auomated build](https://docs.docker.com/docker-cloud/builds/automated-build/#configure-automated-build-settings) to execute whenever I commit to [my git repository](https://github.com/MorganPeat/FirstDockerApp/). This will use my application's [Dockerfile]({% post_url 2018-01-23-docker-aspnet-core-2 %}) to build a Docker image.

Once the build completes I have my completed Docker image, all ready to deploy.
* [My github repo](https://github.com/MorganPeat/FirstDockerApp/)
* [My Docker repository](https://cloud.docker.com/swarm/morganpeat/repository/docker/morganpeat/firstdockerapp/general)

## Deploying to AKS
I mentioned before that I wanted to look at [Kubernetes for service orchestration]({% post_url 2018-01-15-what-is-needed %}) so decided to use it to run my test container. This is very easy to do with [Azure Container Service (AKS)](https://azure.microsoft.com/en-gb/services/container-service/) and all I had to do was follow their straightforward Quick Start tutorial, [Deploy an Azure Container Service (AKS) cluster](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal). Instead of deploying their Docker image I changed the manifest file to point to [mine](https://cloud.docker.com/swarm/morganpeat/repository/docker/morganpeat/firstdockerapp/tags) and, hey presto, a multi-node Cloud application!

Next up, a deeper dive into Kubernetes.

<br/>
## tl;dr
Commit to github, build and host on Docker Cloud, deploy on AKS. Easy, peasy!
