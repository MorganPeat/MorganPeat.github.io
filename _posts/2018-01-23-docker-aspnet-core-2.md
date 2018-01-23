---
layout: post
title:  "First use of Docker with ASP.Net Core part 2"
date:   2018-01-23 12:00:00
categories: [Docker]
tags: [docker, netcoreapp]
draft : false
---
In my [previous post]({% post_url 2018-01-22-docker-aspnet-core %}) I wrote a basic Dockerfile to automate the build of my Docker image. The build failed, so now I need to fix it and get my first container up and running.
<br/>
The build error message was:
{% highlight shell %}
/usr/share/dotnet/sdk/2.1.4/NuGet.targets(103,5): error : Unable to load the service index for source https://api.nuget.org/v3/index.json. [/app/FirstDockerApp.csproj]
/usr/share/dotnet/sdk/2.1.4/NuGet.targets(103,5): error :   An error occurred while sending the request. [/app/FirstDockerApp.csproj]
/usr/share/dotnet/sdk/2.1.4/NuGet.targets(103,5): error :   Couldn't resolve host name [/app/FirstDockerApp.csproj]
{% endhighlight %}
The key part here is `Couldn't resolve host name`. This means that Docker can't access the internet through the corporate firewall. This is the same problem [I had before]({% post_url 2018-01-19-docker-toolbox %}), and can be solved again by setting up the proxy server details. But I found quite a neat way of testing this, and other, fixes to Dockerfiles.

## Interactive Docker container
I can edit my Dockerfile and comment out everything after the first failing statement (`dotnet restore`), then run the docker container [interactively](https://docs.docker.com/engine/reference/run/#foreground) by passing `-it` as a parameter to `docker run`. This gives an interactive shell within the container and allows for testing / debugging of Dockerfile commands. Very powerful!
Within the shell I can add my `export HTTP_PROXY=` settings, run `dotnet restore` and verify that it all works.

{% highlight shell %}
$ docker run -it --rm <container_id>
root@aaaaaaaaaaaa:/app# export HTTP_PROXY=user:password@your.proxy.name:8080
root@aaaaaaaaaaaa:/app# export HTTPS_PROXY=user:password@your.proxy.name:8080
root@aaaaaaaaaaaa:/app# dotnet restore
  Restoring packages for /app/FirstDockerApp.csproj...
  Installing Microsoft.AspNetCore.SpaServices 2.0.1.
  Installing Microsoft.AspNetCore.NodeServices 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.TagHelpers 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.RazorPages 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.ViewFeatures 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.Formatters.Json 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.DataAnnotations 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.Formatters.Xml 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.Razor 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.Razor.ViewCompilation 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.Cors 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.Localization 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.Abstractions 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.ApiExplorer 2.0.1.
  Installing Microsoft.AspNetCore.Mvc 2.0.1.
  Installing Microsoft.AspNetCore.Mvc.Core 2.0.1.
  Installing Microsoft.AspNetCore.All 2.0.3.
  Generating MSBuild file /app/obj/FirstDockerApp.csproj.nuget.g.props.
  Generating MSBuild file /app/obj/FirstDockerApp.csproj.nuget.g.targets.
  Restore completed in 3.82 sec for /app/FirstDockerApp.csproj.
  Restoring packages for /app/FirstDockerApp.csproj...
  Installing Microsoft.VisualStudio.Web.CodeGeneration.Contracts 2.0.1.
  Installing Microsoft.VisualStudio.Web.CodeGeneration.Tools 2.0.1.
  Restore completed in 1.87 sec for /app/FirstDockerApp.csproj.
{% endhighlight %}  

## Proxy settings in Dockerfile

A quick bit of googling shows I can pass these environment variables via the Dockerfile:
* https://stackoverflow.com/questions/46679808/cant-set-proxy-in-dockerfile
* https://stackoverflow.com/questions/22179301/how-do-you-run-apt-get-in-a-dockerfile-behind-a-proxy

It's not ideal to have my username and password hardcoded in a Dockerfile, obviously! But at least this gets me unblocked for my first app. I'll need to contact the company InfoSec guys to work out what 'proper' production usage should be.

<br/>
My Dockerfile now:
{% highlight shell %}
FROM microsoft/aspnetcore-build:2.0
WORKDIR /app

ENV HTTP_PROXY=user:password@your.proxy.name:8080
ENV HTTPS_PROXY=user:password@your.proxy.name:8080

# copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# copy and build everything else
COPY . ./
RUN dotnet publish -c Release -o out

ENTRYPOINT ["dotnet", "out/FirstDockerApp.dll"]
{% endhighlight %}

<br/>
And it builds! In the next post, I'll try and get it running.

<br/>
<br/>
