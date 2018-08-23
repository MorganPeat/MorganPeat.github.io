---
layout: post
title:  "Microservice spike part 3 - run in GKE"
date:   2018-08-23 12:00:00
categories: [Microservices]
tags: [cloud, k8s]
draft: false
---

This is part 3 in a series of posts where I spike out a cloud microservice app on GCP. In this post I set up a Kubernetes (k8s) cluster in Google Kubernetes Engine (GKE), deploy my app and make it available as an external service.

* [Part 1 - the CryptoTracker app]({% post_url 2018-08-21-cryptotracker-a %}) introduces my prototype microservice app
* [Part 2 - run in GCP]({% post_url 2018-08-21-cryptotracker-b %}) gets the app running in Docker in Google Compute Engine
* [Part 3 - run in GKE]({% post_url 2018-08-23-cryptotracker-c %}) (this page) gets the app running in Google Kubernetes Engine


## Create a new Kubernetes cluster and deploy

Rather than exploit the full power of the `kubectl` command line I used the GKE UI to keep things simple. I created a basic k8s cluster with 2 `small` nodes in `europe-west-2` (London) region, keeping everything else as default.

![Create a new k8s cluster in GKE]({{ site.baseurl }}/images/cryptotracker/k8s-1.png "Create a new k8s cluster in GKE")

Once the cluster was up and running it was trivial to deploy my app: just the docker image name and mongo connection string are required and everything else can be kept as default. The only minor issue I had is that the colon character is not supported in an environment variable name, but the [ASP.NET Core configuration](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/#configuration-by-environment) settings copes with this. Double-underscores are supported on all environments so I just changed `MongoDB:ConnectionString` to `MongoDB__ConnectionString` and it all worked fine.

![Deploy my app on GKE]({{ site.baseurl }}/images/cryptotracker/k8s-2.png "Deploy my app on GKE")



## Expose my app as a k8s service

By default, any deployment is not exposed outside the GKE cluster and a service is required to give a public endpoint. When I view my deployment in the Cloud console (`Kubernetes Engine -> Workloads -> ctmd`) I can see a single pod running and have the option to scale it up or down. The pod itself is transient and may be replaced at any time, resulting in a new IP address. A service will have a static IP address and load balance between any number of underlying, transient, pods.

![Expose my app as a service]({{ site.baseurl }}/images/cryptotracker/k8s-3.png "Expose my app as a service")

I have mapped my container port 8080 to port 80 (http default) on my service so I don't need to specify a port in the url any more. Once the service is created I can hit the service's external IP address and access the URL (`http://<service ip>/api/v1/currencies`) exposed by my app!
