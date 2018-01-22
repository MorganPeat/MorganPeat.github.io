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

The Dockerfile is pretty self-explanatory, although I haven't yet worked out why the nuget restore and source code copy are separate steps.
<br/>
<br/>
Building my application from the Docker command line runs this Dockerfile and givesme  this output:
{% highlight shell %}
$ docker build
"docker build" requires exactly 1 argument.
See 'docker build --help'.

723254a2c089: Pull complete
abe15a44e12f: Pull complete
409a28e3cc3d: Pull complete
503166935590: Pull complete
09d35e02522b: Pull complete
7b915999d3d4: Pull complete
73625913309a: Pull complete
b457441733d9: Pull complete
2c97b35a16e4: Pull complete
ad5633a9a2eb: Pull complete
Digest: sha256:f905f7873db5fa5b6d67b326f717616595cb675b9e38bd9d307fe22f3769fc16
Status: Downloaded newer image for microsoft/aspnetcore-build:2.0             ]  41.09MB/153.4MB
 ---> 6105426f13e9loading [==>                                                ]  12.96MB/298.4MB
Step 2/7 : WORKDIR /appcomplete
Removing intermediate container 9b24ca1d3909                                  ]  40.55MB/153.4MB
 ---> bf35441e2363loading [==>                                                ]  12.42MB/298.4MB
Step 3/7 : COPY *.csproj ./
 ---> 863158a144a7ing
Step 4/7 : RUN dotnet restore
 ---> Running in 018f7409d033
  Restoring packages for /app/FirstDockerApp.csproj...
/usr/share/dotnet/sdk/2.1.4/NuGet.targets(103,5): error : Unable to load the service index for source https://api.nuget.org/v3/index.json. [/app/FirstDockerApp.csproj]
/usr/share/dotnet/sdk/2.1.4/NuGet.targets(103,5): error :   An error occurred while sending the request. [/app/FirstDockerApp.csproj]
/usr/share/dotnet/sdk/2.1.4/NuGet.targets(103,5): error :   Couldn't resolve host name [/app/FirstDockerApp.csproj]
The command '/bin/sh -c dotnet restore' returned a non-zero code: 1
{% endhighlight %}
