---
layout: post
title:  "First look at OpenShift"
date:   2018-02-07 12:00:00
categories: [Docker]
tags: [docker, netcoreapp, cloud]
draft: false
---
My company has set its (internal) Cloud strategy as:
* Use [Docker](https://www.docker.com/) containers as the standard application packing mechanism
* Use [Kubernetes](https://kubernetes.io/) as the standard server runtime environment
* Use [OpenShift](https://www.openshift.org/) as the supported distribution of Docker and k8s

So, having got a [simple proof of concept app set up](% post_url 2018-02-02-automated-build-git-docker %) using Docker and k8s on Azure, it's time to look into how to get the same app running in-house on OpenShift.  

## .NET Core support in OpenShift
.NET Core 2.0 support was [added to Red Hat](https://www.redhat.com/en/about/press-releases/red-hat-boosts-application-portability-across-hybrid-cloud-net-core-20) (RHEL and OpenShift) in August 2017 and an [OpenShift blog article](https://blog.openshift.com/announcing-net-core-2-0-support-openshift/) describes how to use it. Currently .NET code will run on Linux only, Windows container support is due to be added [some time in 2018](https://redmondmag.com/articles/2017/08/22/red-hat-openshift-windows-containers.aspx).  
(We were planning to target .NET Core 2.0 on Linux as much as possible, so this is fine for us.)

## Source-to-image (S2I)
In order to run a .NET Core 2.0 container in OpenShift there are a number of "[guidelines](https://docs.openshift.com/enterprise/3.0/creating_images/guidelines.html)" that should be followed. The recommended way is to have OpenShift build your code into an image itself, using [Source-to-image (S2I)](https://github.com/openshift/source-to-image). You merely tell OpenShift to create a new application from the .NET Core 2.0 base image, giving the URL of your git repository. It will build, test and create a OpenShift-compatible Docker image for you.  
For example: `oc new-app dotnet/dotnet-20-rhel7~https://github.com/my/git/repo.git`.  

I couldn't get this to work in-house (translation: I ran out of patience setting up CA certificates etc) so I decided to build my own simple Docker image from the OpenShift .NET Core _runtime_ base image.  
Details of the .NET Core S2I images (for .NET Core 1.0, 1.1, 2.0) are on [github](https://github.com/redhat-developer/s2i-dotnetcore). I used the [.NET Core 2.0 "runtime" image](https://github.com/redhat-developer/s2i-dotnetcore/blob/master/2.0/runtime/README.md), which allows you to build an OpenShift-compatible image from pre-built .NET Core binaries, OpenShift won't build anything for you.  

## Building the Docker image
The details are all on the [github link](https://github.com/redhat-developer/s2i-dotnetcore/blob/master/2.0/runtime/README.md), but in a nutshell:
1. Restore nuget packages, ensuring the RHEL runtime is targeted  
`$ dotnet restore -r rhel.7-x64`
2. Build and publish the app, targeting the same framework  
`$ dotnet publish -f netcoreapp2.0 -c Release -r rhel.7-x64 --self-contained false /p:PublishWithAspNetCoreTargetManifest=false` ([more about the ASP.NET Core implicit store](https://docs.microsoft.com/en-us/dotnet/core/deploying/runtime-store#aspnet-core-implicit-store))  
3. Build the Docker image as normal using a modified Dockerfile
```
FROM dotnet/dotnet-20-runtime-rhel7
ADD bin/Release/netcoreapp2.0/rhel.7-x64/publish/. .
CMD [ "dotnet", "FirstDockerApp.dll" ]
```  

A couple things to note here:
* I pulled the base image with `docker pull registry.access.redhat.com/dotnet/dotnet-20-runtime-rhel7` and re-tagged to remove the `registry.access.redhat.com`  
* The base image runs ASP.NET Core on port 8080, not port 80. This is because OpenShift doesn't allow privileged access to containers and any ports < 1024 are unavailable  

## Running locally
The OpenShift-compatible image can be tested locally as normal, I just need to remember to map port 8080 instead of port 80:  `docker run -d --rm -p 3333:8080 first-docker-app`  

## Pushing the image to an OpenShift repository  
First I need to push my image to the local OpenShift registry. I do this by logging into the UI and creating a new "image stream". The UI tells me what commands I need to use to push the image, but in brief:
* Tag the docker image with the appropriate registry and project name  
`$ docker tag first-docker-app your.registry.net:5000/project-name/first-docker-app:0.1`  
* I'm giving an explicit version for the tag rather than relying on `latest` as this seems to be [best practice](https://medium.com/@mccode/the-misunderstood-docker-tag-latest-af3babfd6375)
* The OpenShift UI gives instructions on how to log in to the docker registry  
`$ docker login -p some-open-shift-token -u unused your.registry.net:5000`

## Deploying on OpenShift  
Using the OpenShift UI I created a new project for my image. To deploy my image I found that the `oc` command-line tools were needed; the OpenShift UI's "help" link explains how to download this and log in.  
<br/>
I used the same [k8s manifest file](https://github.com/MorganPeat/FirstDockerApp/blob/master/k8s-all-in-one.yml) which I used when [playing with AKS](% post_url 2018-02-02-automated-build-git-docker %), but I needed to make a few changes:  
* The docker image needs to point to my in-house repository  
* The port numbers in both the deployment and service need to be changed to 8080  
* The service type I had to change from `LoadBalancer` to `ClusterIP`  

This last change means that the service is only reachable inernally, i.e. to the rest of the k8s cluster. I haven't researched too much into it yet, but it seems that it is better to create an internal service, then a [Route](https://docs.openshift.com/enterprise/3.0/architecture/core_concepts/routes.html) ([Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) in k8s) so that it can be reached from the outside world.  

To create the OpenShift deployment, I ran `$ ./oc create -f k8s-all-in-one.yml` and it span up my deployment in OpenShift! I then created a route using the OpenShift UI, and my image is reachable via a browser.


<br/>
## tl;dr
A number of tweaks are needed to run on OpenShift but the same basic principles apply.
