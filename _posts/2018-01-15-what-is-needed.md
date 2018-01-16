---
layout: post
title:  "Requirements for Cloud POC"
date:   2018-01-12 15:00:00
categories: [CloudMigration]
tags: [cloud, cloud-plan]
---

I have been asked to pull together a list of "stuff" that I need from my company in order to progress our Cloud POC. This means jumping ahead a bit to make some assumptions / guesstimates as to what our Cloud architecture will look like. So without any justification or rationale (which will come later, if these assumptions hold) our application will look something like this (stolen) image:
![Application architecture](https://developer.xamarin.com/guides/xamarin-forms/enterprise-application-patterns/containerized-microservices/Images/microservicesarchitecturewitheventbus.png)

## What this means to our application
* Public APIs exposed via REST / HTTP (maybe served via some [Gateway API](https://microservices.io/patterns/apigateway.html))
* Containerised microservices, each hosting its own web interface and having its own storage
* Some common event bus (either RabbitMQ or some Cloud-native bus like Azure Service Bus / Amazon Simple Queue Service)
* MongoDB as the data store (our application is tightly bound to Mongo so we have no real alternative at the moment)
* Service orchestration via Docker Swarm, Kubernetes, etc

Putting more flesh on the image and list above:
### Cloud access
Once we get access to a Cloud vendor we should be able to do what we want. But to get there we need a few things in place:
#### Cloud account
We are attempting to be fairly vendor-agnostic when it comes to migrating to Cloud. Partly because it seems to be good practice, partly because it prevents lock-in to any particular vendor, but mainly because as a single project in a large corporate we have little choice over where our application is hosted. Clearly some things (e.g. message bus) may be easier if we target a specific offering but the more neutral we can be at this stage, the easier it will be when the final vendor decision is made.
But ultimately we will need a Cloud login which we can use to start playing with. Obviously it will need to be backed by someone's credit card, or prepaid corporate account, so we can actually use it.
#### Firewall access
Our firewalls are pretty well locked down and do a good job at stopping anything unauthorised going in or out. So until proper policies are put in place we may need temporary permission to punch some holes through the firewall so we can start pushing code and data.
#### Legal permission
Although we don't have anything particularly sensitive in our application's data, we will need sign-off from someone in the Legal / Information Security groups before we can start sending things outside the company.
#### Permissioning and authentication
I understand it is possible to link a corporate's ActiveDirectory instance to Cloud providers; also that our company is looking to implement its Single Sign-On solution in the Cloud. Either way, we need to hook into those projects in order to secure our app in accordance with corporate guidelines.

### Development environment
We have a pretty good setup at the moment with a fairly automated set of tools (git via BitBucket, Jira, TeamCity, Visual Studio) but we will need more in order to set up containerised services
#### Windows Server 2016
Required in order to run Docker containers, see [Docker for Windows Server](https://www.docker.com/docker-windows-server), and Kubernetes, see [Kubernetes on Windows](https://docs.microsoft.com/en-us/virtualization/windowscontainers/kubernetes/getting-started-kubernetes-windows).
#### Windows 10
Required in order to build, debug and test Docker containers. Microsoft Windows 10 Professional or Enterprise 64-bit is required. See [Docker for Windows](https://www.docker.com/docker-windows)
As a side note it seems that [Docker Toolbox](https://docs.docker.com/toolbox/toolbox_install_windows/) will enable Docker use on older Windows machines although containers won't be supported natively: a small Linux VM will spin up to host a container.
#### CI / CD support
It seems that [TeamCity can support Docker](https://confluence.jetbrains.com/display/TCD10/Integrating+TeamCity+with+Docker) and JFrog Artifactory (which we use in-house to manage packages) can act as a [Docker Registry](https://www.jfrog.com/confluence/display/RTF/Docker+Registry). These tools may suffice for now, but we'll need to review what best practices are as we start developing in anger.


### Dependencies
The above list should give us the ability to develop and deploy our microservices, but there are dependencies which also need to be in place
#### MongoDB
Other teams are looking at [Atlas](https://www.mongodb.com/cloud/atlas), MongoDBs DBaaS offering. If we can ride off the back of that initiative then great! Otherwise it should be straightforward for us to install mongo  on an EC2 instance, there are AMIs in the Amazon Marketplace we could investigate.
#### Message Bus
If we can provide a good abstraction for our application we should be able to swap out our in-house message bus for a Cloud native one, or perhaps a RabbitMQ broker running on a VM. Either way, this is for our team's consideration and not something we need to ask our company to help with.
#### Monitoring and logging
Again this is mostly a concern for our team rather than an 'ask'. Our support team have a tool they use to monitor all their apps and are reluctant to watch several different dashboards. Whatever solution we choose for monitoring, we will need to find a way to plug into this tool.


## The shopping list
In a nutshell, then, here are the things we need to get our company to help provide:
* A Cloud login (vendor does not matter, but it would be nice to have a couple vendors to compare) with the financial backing needed to use it properly
* Firewall permissions to send code and data to and from the Cloud
* Legal / InfoSec sign-off for our proprietary code and data
* Windows Server 2016 for in-house development and testing of Kubernetes and Docker containers
* Windows 10 Professional or Enterprise for Docker development

Less urgent, but required before we could be 'production ready':
* A MongoDB Atlas account with the necessary financial backing, plus the firewall holes and InfoSec permission needed to use it
* A link to the necessary CI / CD tools which can support containers and deploy to Cloud
* A link to the corporate Sigle Sign-On solution or ActiveDirectory instance in order to correctly permission our APIs



## tl;dr
We aim to refactor to containerised microservices using MongoDB and a Cloud-native message bus. We need some £££, support and our shopping list in order to progress further.
