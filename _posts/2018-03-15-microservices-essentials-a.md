---
layout: post
title:  "Microservices essentials part 1 - the platform"
date:   2018-03-15 11:00:00
categories: [Microservices]
tags: [microservices]
draft: false
---

There are a whole host of principles and best practices to follow when moving to microservices but I'm only going to document the main ones which will affect me and the projects I work on. Most of this is copy n' paste from a very good blog article, [Best Practices for Building a Microservice Architecture](http://www.vinaysahni.com/best-practices-for-building-a-microservice-architecture), but re-ordered it to suit my aims.  

* [Part 1 - the platform]({% post_url 2018-03-15-microservices-essentials-a %}) (this page) introduces the microservice platform, development set-up and how to identify service boundaries.
* [Part 2 - platform design]({% post_url 2018-03-15-microservices-essentials-b %}) covers the design of the overall platform (data, state, consistency, communication and serialisation protocols)
* [Part 3 - the service]({% post_url 2018-03-15-microservices-essentials-c %}) covers technical aspects to be considered for an individual service (API versioning, retry and failure policies, feature flags)

## The microservice platform
A microservice architecture shifts around complexity. Instead of a single complex system you have a bunch of simple services with complex interactions. The aim is to keep the complexity in check.  

You need to set some standards to ensure consistency and to manage operational complexity. This means standardising communications, logging, monitoring and deployment, among others. Your platform is a set of standards combined with tools that makes it easy to create and operate services that meet the standards.

Within your platform, you'll be running many services (tens, hundreds or even thousands). Each service encapsulates a piece of business capability into an independent package. You need to build services that are small enough to keep them focused on a single purpose yet big enough to minimise interactions with other services.

## Development Set-up

### Source control
Each service should have its own repository. This keeps checkouts small, source control logs clean and enables granular access control. You're not deploying services together and shouldn't be co-locating their code either.

### Development environments
Given the complexity and number of services involved in a microservice approach it may not be practical to bring everything up on a single developer machine. In that case, the service that is developed and running locally could be combined with an isolated environment running in the cloud. The developer would be able to quickly iterate in their development environment while testing against other services running in the cloud. Note that isolation is critical for such cloud environments. Shared environments between developers will only cause confusion as a result of unexpected changes.


## Identifying Service Boundaries
Each service should be an autonomous unit that implements a business capability.

A service should be **loosely coupled**. It should have minimal dependence on other services. Any communication it does with other services should be over the exposed public interfaces (API, events, etc). Those interfaces also need to be designed to not expose internal details.  

A service should have **high cohesion**. Closely related functionality should stay together in the same service. This minimizes chattiness between services.

A service should cover a **single bounded context**. A bounded context encapsulates internal details of a domain, including any domain specific models.

Ideally, you understand your product and business well enough to have identified natural service boundaries. Even if you get it wrong the first time around, loose coupling makes it easy to refactor (i.e. combine, split or restructure) services in the future.  

#### Service size

*Micro* in microservice has nothing to do with the physical size or lines of code, it's about minimizing complexity. A service should be *small enough* that it serves a focused purpose. At the same time, it should be *big enough* that it minimizes interservice communication.

There's no hard set rule that a service can only be one process, one virtual machine, or one container. A service consists of what it needs to autonomously implement a business capability. This includes external services like data servers for persistence, job queues for asynchronous workers or even caches to keep things fast.

## Validating the design
Excerpt from a Microsoft Azure article on [identifying microservice boundaries](https://docs.microsoft.com/en-us/azure/architecture/microservices/microservice-boundaries).  

After you identify the microservices in your application, validate your design against the following criteria:
* Each service has a single responsibility.
* There are no chatty calls between services. If splitting functionality into two services causes them to be overly chatty, it may be a symptom that these functions belong in the same service.
* Each service is small enough that it can be built by a small team working independently.
* There are no inter-dependencies that will require two or more services to be deployed in lock-step. It should always be possible to deploy a service without redeploying any other services.
* Services are not tightly coupled, and can evolve independently.
* Your service boundaries will not create problems with data consistency or integrity. Sometimes it's important to maintain data consistency by putting functionality into a single microservice. That said, consider whether you really need strong consistency. There are strategies for addressing eventual consistency in a distributed system, and the benefits of decomposing services often outweigh the challenges of managing eventual consistency.  

Above all, it's important to be pragmatic, and remember that domain-driven design is an iterative process. When in doubt, start with more coarse-grained microservices. Splitting a microservice into two smaller services is easier than refactoring functionality across several existing microservices.
