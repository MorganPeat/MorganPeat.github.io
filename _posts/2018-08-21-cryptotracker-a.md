---
layout: post
title:  "Microservice spike part 1 - the CryptoTracker app"
date:   2018-08-21 13:00:00
categories: [Microservices]
tags: [microservices, cloud, docker, netcoreapp]
draft: false
---

This is part 1 in a series of posts where I spike out a cloud microservice app on GCP. In this post I describe the design of my simple ASP.NET Core microservice and the patterns I have followed which will help it to work well in the Cloud.

* [Part 1 - the CryptoTracker app]({% post_url 2018-08-21-cryptotracker-a %}) (this page) introduces my prototype microservice app
* [Part 2 - run in GCP]({% post_url 2018-08-21-cryptotracker-b %}) gets the app running in Docker in Google Compute Engine



## A simple .NET Core app: CryptoTracker

I have pushed up to github a pretty simple microservice application which I will use to help learn 'cloud stuff'. It is deliberately basic at the moment since for this spike I'm more interested in Cloud infrastructure than application code. The app stores a short list of cryptocurrencies in a mongo DB and serves the it up in json format via a HTTP GET from ASP.NET Core.

The code for this article is available at [https://github.com/MorganPeat/CryptoTracker/tree/v1](https://github.com/MorganPeat/CryptoTracker/tree/v1)

### Logging

Best practice when logging from a docker container seems simply to be: write to `stdout` / `stderr`. The docker runtime will hook into the console to pick the logs and forward them on to wherever you want. This keeps application logging config / dependencies simple and enables you to change the logging destination without altering the docker image: I can log to a file when running locally or push to Splunk / Stackdriver / wherever when running in the Cloud.

> Ideally, applications log to stdout/stderr, and Docker sends those logs to the configured logging destination.

[https://success.docker.com/article/logging-best-practices](https://success.docker.com/article/logging-best-practices)

> The Docker logging driver reads log events directly from the container’s stdout and stderr output; this eliminates the need to read to and write from log files.

[http://www.monitis.com/blog/containers-5-docker-logging-best-practices/](http://www.monitis.com/blog/containers-5-docker-logging-best-practices/)

I have configured logging so it is routed through [Serilog](https://serilog.net/) using [Serilog.AspNetCore](https://github.com/serilog/serilog-aspnetcore). This ensures all application code and ASP.NET Core internal code logs the same way. I'm logging to the console in json format in the hope that I can get a Cloud logging provider (e.g. Stackdriver) to parse the json and pick up message, severity, etc.

I also added the `LoggingEvents.cs` class in order to include a [Log event ID](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/#log-event-id) with certain log messages. This will make it easier to search for relevant log lines.


### Configuration

There are multiple ways of picking up application configuration in a container. [This](https://dantehranian.wordpress.com/2015/03/25/how-should-i-get-application-configuration-into-my-docker-containers/) article, while old, is [still relevant](https://stackoverflow.com/questions/48183023/how-should-i-get-application-configuration-into-my-docker-containers). In brief, the main options are:

1. Baking the configuration into the container
1. Setting the application configuration dynamically via environment variables
1. Setting the application configuration dynamically via environment variables (using an external KV store)
1. Map the config files in directly via docker volumes

Option 1 is generally considered to be bad practice since (a) secrets are embedded in the image and (b) a new image needs to be published if any secrets change. Options 2 and 3 are a bit complicated for me right now (it's only a spike) so I'm going for option 2.  Handily, this is well supported in the .NET Core world where you can use json files to [configure ASP.NET Core per environment](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/#configuration-by-environment).   

In my app I have created a base `appsettings.json` file which contains my (only) configuration key: a mongo db connection string. By default this setting is empty. I have done this to (a) document which settings the app expects, and (b) to fail fast if a required setting is missing.  To allow local F5-style running and debugging I have added a `.gitignore`d `appsettings.Development.json` file containing a connection string to my local mongo. When running in docker I can hedge my bets and choose the most appropriate option for the infrastructure I deploy to:

*Inject configuration through environment variables*

In `Startup.cs` I call `.AddEnvironmentVariables();` at the end of the configuration chain. If a setting is injected via an environment variable it will override any previously loaded settings. Exactly how this is done (i.e. Option 2 or 3) doesn't matter to my image as long as the required environment variables are present.

*Map config files via docker volumes*

I haven't tried this, since I am going the Environment Variable route, but it should now be possible to map a `appsettings.Production.json` file into my container and pick up any settings that way. In `Startup.cs` I call `.AddJsonFile(Path.Combine("config", $"appsettings.{env.EnvironmentName}.json"), optional: true)`.  Here the `ASPNETCORE_ENVIRONMENT` environment variable is used to determine which configuration file should be loaded (Development, Production, etc) and, since it's optional, my app won't break if a file is missing. Note the use of `Path.Combine`: this is to cater for different path separators when running on Linux vs Windows (`/` vs `\`).`



### Data layer abstraction

Although it may be overkill for such a simple app I have pushed all interaction with mongo db behind a layer of abstraction (kind of like the [Repository pattern](https://martinfowler.com/eaaCatalog/repository.html)). It's pretty half-baked, being a small spike app, but I like the idea of abstracting all mongo logic away so I can change my storage tier easily.

I'm also playing with the idea of a kind of Entity Framework-less [Code First](http://www.entityframeworktutorial.net/code-first/what-is-code-first.aspx) approach to storing my data in mongo. When the app starts up it configures the database: seed data, indexes and so on. I'm not sure what the 'best practice' approach would be to this kind of thing yet but it suffices to get my db in the right state for the app to run.



# Dockerfile

My docker build script is in `dockerfile`. (There's also a `dockerfile-internal` which includes instructions for my internal nuget repository, SSL certificates, etc but this is `.gitignore`d in order to keep internal data away from github). I'm using a [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) to keep the final image size down. The transient build image is based off the [.NET Core SDK base image](https://hub.docker.com/r/microsoft/dotnet/) and the final image uses the ASP.NET Core base image for it's smaller size.  

In an attempt to leverage the docker [build cache](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache) the dockerfile is ordered: instructions I don't expect to change much come first in order to hit the build cache: base images, configuration, nuget restore. Source files (which will change more frequently than their parent `csproj`) are added as late in the process as possible. This seems to be the approach used in many [examples](https://docs.docker.com/engine/examples/dotnetcore/), so my working assumption is that this is a good thing to do.

I have hard-coded the `ASPNETCORE_ENVIRONMENT` to `Production` although I guess this could be overriden by injecting an environment variable. I don't think this variable has much use at the moment: I'm not using `appsettings.<Environment>.json` at the moment. I guess a developer exception page could be nice, but by default I'd rather leave it as production.

The ASP.NET Core kestrel site has been set to run on port 8080 via `ENV ASPNETCORE_URLS http://*:8080` and `EXPOSE 8080`. This allows the configuration to be static, therefore simpler (no need to work out the port from environment config, etc). When the container is executed the internal [port will need to be mapped](https://docs.docker.com/v17.09/engine/userguide/networking/default_network/binding/) to a port on the host machine anyway.



### Other build stuff

* `.gitignore` has been set up to ignore sensitive internal files (`nuget.config`, SSL certificates, development app settings)
* `.dockerignore` excludes any R#, git, or Visual Studio nonsense when building my image. This keeps the docker [build context](https://docs.docker.com/engine/reference/commandline/build/#extended-description) size down, giving quicker builds
