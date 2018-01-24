---
layout: post
title:  "First use of Docker with ASP.Net Core part 3"
date:   2018-01-24 12:00:00
categories: [Docker]
tags: [docker, netcoreapp]
draft : false
---
In my [previous post]({% post_url 2018-01-23-docker-aspnet-core-2 %}) I got my basic Dockerfile to build. Now I can run it and prove my first Docker application! First I'll make a couple changes to the build process, then I'll run the container and prove that it is running.

## Docker build changes

#### Build-time variables
I used the `ENV` keyword in my Dockerfile to set the HTTP proxy. This isn't ideal since I only want this variable at build time, not in my final image. I guess I could use [multi-stage builds](https://docs.docker.com/engine/userguide/eng-image/multistage-build/) to ensure the variables are in the build image, not the final release one, but a nicer solution is to have transient variables that are present at build time and not persisted in the final image. [Build-time variables](https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables-build-arg) do this.  

`$ docker build --build-arg HTTP_PROXY=user:password@your.proxy.name:8080 .`
<br/>

#### Tagged images
I can also give my image a nice name by [tagging](https://docs.docker.com/engine/reference/commandline/build/#tag-an-image--t) it.
`$ docker -t first-docker-app`
<br/>

My docker build command now looks like this:  
`$ docker build -t first-docker-app --build-arg HTTP_PROXY=user:password@your.proxy.name:8080 --build-arg HTTPS_PROXY=user:password@your.proxy.name:8080 .`
<br/>

## Starting the container
I can start my container now, and see it running!
`$ docker run -d -e "ASPNETCORE_URLS=http://+:80" -p 8000:80 my-first-app`  

The parameters here are
* `-d` to run in [detached](https://docs.docker.com/engine/reference/run/#detached-vs-foreground) (i.e. background) mode
* `-e "ASPNETCORE_URLS=http://+:80"` to set a container [environment variable](https://docs.docker.com/engine/reference/run/#env-environment-variables) which will [tell ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/hosting?tabs=aspnetcore2x) to listen to any incoming requests on port 80.
* `-p 8000:80` to [expose an incoming port](https://docs.docker.com/engine/reference/run/#expose-incoming-ports) on the container, mapping port 80 on the container to port 8000 on the host. (I'm running Docker Toolbox so, in this case, the "host" is my VM)

As an aside, it seems that the [best practice](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#expose) when writing a Dockerfile is to use the "common, traditional" port for an application. So a web server would listen on port 80, a Mongo DB would listen on port 27017, etc.

## VM IP address and NAT
Because I'm running Docker Toolbox on Windows 7, my docker container is hosted in a VM. That means my app is not listening on `localhost`, but on the VM.  
`$ docker-machine ip` gives me the IP address of my VM; I can then browse to `http://<vm-ip>/api/values` and see it working.  
<br/>  
In order to access this page _outside_ my machine, I need to forward port 8000 on my host to port 8000 on my VM. I can do this through the Oracle VM VirtualBox Manager UI:
1. Open Oracle VM VirtualBox Manager
2. Select the VM used by Docker
3. Click Settings -> Network
4. Adapter 1 should (default?) be "Attached to: NAT"
5. Click Advanced -> Port Forwarding
6. Add rule: Protocol TCP, Host Port 8080, Guest Port 8080 (leave Host IP and Guest IP empty)
7. Guest is your docker container and Host is your machine

More details, and a great write-up on this, at https://stackoverflow.com/questions/33814696/how-to-connect-to-a-docker-container-from-outside-the-host-same-network-windo
