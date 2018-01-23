---
layout: post
title:  "Deciding to move to microservices"
date:   2018-01-12 15:00:00
categories: [Microservices]
tags: [microservices]
draft : false
---

We have decided to refactor our application to a Microservice architecture because we think it will be easier to migrate to Cloud, support and maintain. This means, in the language of Martin Fowler, is
<blockquote>
an approach to developing a single application as a suite of small services, each running in its own process and communicating with lightweight mechanisms, often an HTTP resource API. These services are built around business capabilities and independently deployable by fully automated deployment machinery. There is a bare minimum of centralized management of these services, which may be written in different programming languages and use different data storage technologies.
</blockquote>
https://martinfowler.com/articles/microservices.html

Instead of putting all the functionality into a single block and scaling out by replicating across servers, each element of functionality becomes a separate service which is deployed in isolation, and the service can be scaled out by replicating as required.
![Decentralised data](https://martinfowler.com/articles/microservices/images/decentralised-data.png)


## Why bother?
Links to useful resources explaining microservices, their benefits and drawbacks
* https://martinfowler.com/articles/microservices.html
* https://martinfowler.com/microservices/
* https://martinfowler.com/articles/microservice-trade-offs.html
* http://www.vinaysahni.com/best-practices-for-building-a-microservice-architecture This article contains a whole bunch of best practices which I'll be looking into in more detail soon
* https://www.codementor.io/murphyisiwele/microservices-vs-monolithic-applications-why-you-should-consider-microservices-b2jdwgz2l


## tl;dr
We are moving to microservices, there's a whole lot of stuff to think about
