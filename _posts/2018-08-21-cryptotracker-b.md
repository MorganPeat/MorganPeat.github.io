---
layout: post
title:  "Microservice spike part 2 - run in GCP"
date:   2018-08-21 15:00:00
categories: [Microservices]
tags: [microservices, cloud, docker, netcoreapp]
draft: false
---

This is part 2 in a series of posts where I spike out a cloud microservice app on GCP. In this post I set up a Cloud MongoDB cluster, provision a Google Compute Engine VM and deploy my simple app in a docker container.

* [Part 1 - the CryptoTracker app]({% post_url 2018-08-21-cryptotracker-a %}) introduces my prototype microservice app
* [Part 2 - run in GCP]({% post_url 2018-08-21-cryptotracker-b %}) (this page) gets the app running in Docker in Google Compute Engine
* [Part 3 - run in GKE]({% post_url 2018-08-23-cryptotracker-c %}) gets the app running in Google Kubernetes Engine
* [Part 4 - run in an Istio service mesh]({% post_url 2018-08-30-cryptotracker-d %}) gets the app running in an Istio service mesh
* [Part 5 - secure the app via TLS]({% post_url 2018-09-12-cryptotracker-e %}) secures the app by using TLS over HTTPS



## Run in Docker

The easiest way to make my docker image available to Google Cloud Platform (GCP) is to push it to [Docker Hub](https://docs.docker.com/docker-hub/). I [blogged about this]({% post_url 2018-02-02-automated-build-git-docker %}) already so I won't cover it here, suffice to say that every time I push to my [github repo](https://github.com/MorganPeat/CryptoTracker) Docker Hub will build my image for it and [make it available](https://hub.docker.com/r/morganpeat/cryptotracker/) in its cloud repository.



## Cloud MongoDB

It is amazingly easy to provision a Cloud MongoDB. I created a [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) account then created a new cluster in a local GCP zone. I used the free tier since I'm just playing, which limits the other options available.

![Create a new cluster in MongoDB Atlas]({{ site.baseurl }}/images/cryptotracker/atlas-config-1.png "Create a new cluster in MongoDB Atlas")

Once the cluster has spun up the `Connect` button shows that, by default, connections will be accepted from any IP address (IP whitelist of `0.0.0.0/0`) and it will give me the connection string to use for my application.

![Connect to Atlas cluster]({{ site.baseurl }}/images/cryptotracker/atlas-config-2.png "Connect to Atlas cluster")

In the `Security` tab I added a new user with "Read and write to any database" permission. The above connection string was modified with the correct username and password (instead of `admin:<Password>`) and authentication database (`admin` instead of `test`).  



## Provisioning a VM

After logging into [Google Cloud Console](https://console.cloud.google.com) (I'm using the [free trial](https://cloud.google.com/free/) here too so there's nothing to pay) I can provision a VM to run my container.    I want to keep things simple for now so I'm provisioning a simple VM image in [Compute Engine](https://cloud.google.com/compute/docs/containers/) and will run docker within it.  I created a new VM with the following parameters:

![Provision a new VM]({{ site.baseurl }}/images/cryptotracker/vm-config.png "Provision a new VM")

* I chose a small VM size (`micro`) since this is only a test
* Choosing "Deploy a container image to this VM instance" defaults the VM to Google's [Container-Optimized OS](https://cloud.google.com/container-optimized-os/docs/concepts/features-and-benefits)
* Since my image is deployed to Docker Hub I only need provide the image name in order for it to be deployed
* I want HTTP traffic to be able to hit my container. (I'll look into networking / firewalls at a later date)

The only other thing to do is set the mongo connection string. As discussed in [my last blog article]({% post_url 2018-08-21-cryptotracker-a %}) I can inject config via environment variables. Under "advanced container options" I add an environment variable `MongoDB:ConnectionString` and set it to the connection string extracted from MongoDB Atlas.


## Firewall

Once the VM is created I can view the logs using the `Stackdriver logging` link within the VM details screen. Once I see the log message
> Now listening on: http://[::]:8080

...I know that the VM started up, pulled my docker image, started up and connected to MongoDB Atlas. However I can't connect to it yet because my container is listening on port 8080 and that isn't permitted through the firewall.  

In the VM instance details I can see that (because I ticked the `Allow HTTP traffic` box) it has been given a Network Tag of `http-server`. In the Cloud Console, under VPC network -> Firewall rules I can create a new rule that will allow any IP address (source address of `0.0.0.0/0`) to connect to any machine in my project's network with that tag on TCP port 8080.

![Create a firewall rule]({{ site.baseurl }}/images/cryptotracker/firewall.png "Create a firewall rule")

Once that has created, I can navigate to `http://external-ip-address:8080/api/v1/currencies` and see my list of cryptocurrencies stored in Mongo!
