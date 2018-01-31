---
layout: post
title:  "Microsoft Docker images"
date:   2018-01-19 15:00:00
categories: [Docker]
tags: [docker, netcoreapp]
draft : false
---
There are a whole host of Docker base images created by Microsoft so for my own benefit I have summarised them here.

## Multi-stage builds
[Docker best practices](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#use-multi-stage-builds) suggest using multi-stage builds when creating images. In the dotnet world this means using an intermediate "build" image (with associated SDKs, etc) to compile and build your app; then copying to a "final" image which is based off a more optimised, bare-bones install. Aside from the more lightweight base image it means you avoid all the hassle of the intermediate files ("obj" etc): either deleting them or shipping an overblown final image.

## microsoft/dotnet
* [https://hub.docker.com/r/microsoft/dotnet/](https://hub.docker.com/r/microsoft/dotnet/)
* [https://github.com/dotnet/dotnet-docker](https://github.com/dotnet/dotnet-docker)
* [https://docs.microsoft.com/en-us/dotnet/core/docker/building-net-docker-images](https://docs.microsoft.com/en-us/dotnet/core/docker/building-net-docker-images)

These are the official images for .NET Core for
* Linux
* Windows Server 2016 Nano Server
<br/>

For each variation (see below) there are images targeting different dotnet core versions (1.0, 1.1, 2.0) and different runtimes (Linux amd64, Windows Server 2016 v1709 (Fall Creators Update) amd64, Windows Server 2016 amd64, Linux arm32).
<br/>

The variations here are:
#### microsoft/dotnet:<version>-sdk (e.g. microsoft/dotnet:2.0.0-sdk)
This image contains the dotnet SDK: the dotnet core and command-line CLI tools. It's the one you would use to build a final image with or to run in debug, development or UAT environments.
#### microsoft/dotnet:<version>-runtime
This image contains the dotnet core and runtime; it's optimised for running in production environments.
#### microsoft/dotnet:<version>-runtime-deps
This contains the native dependencies needed to run dotnet core apps, but not the dotnet core itself.
<br/>
The `runtime` vs `runtime-deps` decision depends on whether an application is "framework-dependent" or "self-contained". See [here](https://docs.microsoft.com/en-us/dotnet/core/deploying/index) for more details but, in a nutshell, a framework-dependent application requires the dotnet core runtime and starts like `dotnet MyApplication.dll`. A self-contained application can run without dotnet core installed and starts like `MyApplication.exe`.
<br/>


## microsoft/aspnetcore, microsoft/aspnetcore-build
* [https://hub.docker.com/r/microsoft/aspnetcore/](https://hub.docker.com/r/microsoft/aspnetcore/)
* [https://hub.docker.com/r/microsoft/aspnetcore-build/](https://hub.docker.com/r/microsoft/aspnetcore-build/)
* [https://github.com/aspnet/aspnet-docker](https://github.com/aspnet/aspnet-docker)

These are the official images for building and running compiled ASP.NET Core applications. As you would expect, the ASP.Net Core images are based on the .NET Core base images.
In the runtime image (`microsoft/aspnetcore`) each specific runtime image contains native ASP.NET Core binaries for improved cold start performance (apparently JIT-ing the runtime is very slow). The build image (`microsoft/aspnetcore-build`) contains a local nuget cache for the ASP.NET Core binaries to improve `dotnet restore` performance.
