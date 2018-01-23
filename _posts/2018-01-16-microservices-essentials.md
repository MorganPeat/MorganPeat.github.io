---
layout: post
title:  "Microservices essentials"
date:   2018-01-16 17:00:00
categories: [Microservices]
Xtags: [microservices]
tags: [draft]
draft: true
---

There are a whole host of principles and best practices to follow when moving to microservices, and it is worth documenting the main ones which will affect me and the projects I work on. Most of this is taken from a very good blog article, [Best Practices for Building a Microservice Architecture](http://www.vinaysahni.com/best-practices-for-building-a-microservice-architecture), and I have just summarised the key points here.


Martin Fowler has a lot of good advice about things to think about before moving to microservices. The term he coined is [Microservice Envy](https://www.thoughtworks.com/radar/techniques/microservice-envy), the temptation to rush into microservices without considering the complexity premium that must be paid and the procedures that need to be in place. In the hope of avoiding as many mistakes as possible, here are some of the things we need to think about.


## Microservice Premium
Microservice architectures don't come cheap, there is a premium to be paid due to the additional complexities of monitoring, eventual consistency, automated deployments, etc.
Fowler's advice is *don't even consider microservices unless you have a system that's too complex to manage as a monolith*.
![Microservice premium](https://martinfowler.com/bliki/images/microservice-verdict/productivity.png)
https://martinfowler.com/bliki/MicroservicePremium.html


## Prerequisites
Fowler suggests that, given the additional complexity supporting a microservice application (i.e. a whole bunch of small services rather than just a few), certain competencies should be in place before even considering a move to microservice architecture.
![Prerequisites](https://martinfowler.com/bliki/images/microservicePrerequisites/sketch.png)
### Rapid provisioning
The ability to spin up a new machine in a matter of hours. Clearly this is impossible in a cumpersome corporate, but should easily be possible in the Cloud.
### Basic monitoring
Due to the number of loosely connected services a good monitoring regime will need to be in place to spot problems quickly.
### Rapid application deployment
i.e. the ability to fix problems quickly.

So, basically, a good DevOps culture needs to be in place.

Fowler suggests creating and deploying just a handful of microservices to start with, and leaving the rest of the application as-is. This will give a soft transition and give time to adapt and tweak the architecture.

https://martinfowler.com/bliki/MicroservicePrerequisites.html


## Monolith first strategy
Although not a hard and fast rule (i.e. YMMV) another of Fowler's guidlines is to build a monolith first then gradually move to microservices, because YAGNI. In new, relatively simple apps the burden of microservices will slow cadence and bring unnecessary complexity.
The approaches then are:
### Define boundaries carefully
Build the monolith with well-defined modules for both the APIs and data storage. It should be relatively simple to move to microservices from here. (FWIW I think this is where our aplication is now)
### Peel off microservices
Go for low-hanging fruit first, then see what is left behind. The core _should_ be relateively stable with most development in the new microservices. (Again, I think this may be the most obvious approach for our application).
### Replace the monolith entirely
Yikes!
### Start with coarse-grained services
This gets you used to dealing with  microservices in the wild. Larger services can then be broken down into finer-grained units.

https://martinfowler.com/bliki/MonolithFirst.html


## tl;dr
Move to microservices only when your application is too big, and move gradually. A good DevOps culture needs to be in place. Start with a monolith then gradually peel off microservices.
