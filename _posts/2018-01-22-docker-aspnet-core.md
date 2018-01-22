---
layout: post
title:  "First use of Docker with ASP.Net Core"
date:   2018-01-19 15:00:00
categories: [Docker]
tags: [docker, netcoreapp]
---
Now I have Docker [up and running]({% post_url 2018-01-19-docker-toolbox %}) I wanted to set up an ASP.Net Core application. As with Docker itself, getting a simple "Hello, World" application running in the corporate environment was not as straightforward as I could have hoped. Below is the summary of the steps I had to take which will hopefully save other people some considerable time! I have pushed the source code up to [my git repository](https://github.com/MorganPeat/FirstDockerApp/) and have made each incremental change a distinct commit so it is easy to follow.

## Hello, world
First I started off with a very basic ASP.Net Core app by creating a new project in Visual Studio and accepting all the defaults. I didn't add Docker support since I want to incrementally add it rather than jumping straight in with Docker Compose etc.
[My initial commit](https://github.com/MorganPeat/FirstDockerApp/commit/beb9afa78b5b2c1fb21974b8b5ffea2e33ff1dcb) shows the basic app which can be run and tested via standard dotnet:
{% highlight csharp %}
dotnet build
dotnet run
{% endhighlight %}
Browse to http://localhost:53545/api/values

<br/>
## Dockerfile
In order to build a Docker image in a repeatable, automated way, we can use a [Dockerfile](https://docs.docker.com/engine/reference/builder/). This is a simple text document which starts with a base image (like a basic [Ubuntu install](https://hub.docker.com/_/ubuntu/) or [Windows Nanoserver](https://hub.docker.com/r/microsoft/nanoserver/)) followed by a list of command-line instructions that assemble your image. A simple Dockerfile would look something like this one, a mix of two dotnet sample Dockerfiles for [basic dotnet](https://github.com/dotnet/dotnet-docker-samples/blob/master/dotnetapp-dev/Dockerfile) and [aspnet](https://github.com/dotnet/dotnet-docker-samples/blob/master/aspnetapp/Dockerfile):

{% highlight shell %}
FROM microsoft/aspnetcore-build:2.0
WORKDIR /app

# copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# copy and build everything else
COPY . ./
RUN dotnet publish -c Release -o out

ENTRYPOINT ["dotnet", "out/firstdockerapp.dll"]
{% endhighlight %}
