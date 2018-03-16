---
layout: post
title:  "Microservices essentials part 2 - platform design"
date:   2018-03-15 12:00:00
categories: [Microservices]
tags: [microservices]
draft: false
---

There are a whole host of principles and best practices to follow when moving to microservices but I'm only going to document the main ones which will affect me and the projects I work on. Most of this is copy n' paste from a very good blog article, [Best Practices for Building a Microservice Architecture](http://www.vinaysahni.com/best-practices-for-building-a-microservice-architecture), but re-ordered it to suit my aims.  

* [Part 1 - the platform]({% post_url 2018-03-15-microservices-essentials-a %}) introduces the microservice platform, development set-up and how to identify service boundaries.
* [Part 2 - platform design]({% post_url 2018-03-15-microservices-essentials-b %}) (this page) covers the design of the overall platform (data, state, consistency, communication and serialisation protocols)
* [Part 3 - the service]({% post_url 2018-03-15-microservices-essentials-c %}) covers technical aspects to be considered for an individual service (API versioning, retry and failure policies, feature flags)

## Independently developed and deployed  
No coordination should be needed with other service teams if no breaking API changes have been made. Each service is effectively its own product with its own codebase and lifecycle.

* If you find the need to deploy services together, you're doing something wrong
* If you have a single codebase for all your services, you're doing something wrong
* If you have to send out a warning before each deploy of a service, you're doing something wrong

**Watch out for shared libraries!** If changes to a shared library require all services be updated simultaneously, then you have a point of tight coupling across services. Carefully understand the implications of any shared library you're introducing.

## Private data
Once multiple services are directly reading and writing the same tables in a database, any changes to those tables requires coordinated deployment of all those services. This goes against our prime directive of service independence. Sharing data storage is a sneaky source of coupling.  

Each service needs its own database, possibly co-located within a shared data server. The key point it that the services should have no knowledge of each other's underlying database. This way, you can start out with a shared data server and separate things out in the future with just a config change.

However, sharing a data server does have its own complications. Firstly, it becomes a single point of failure that can take down a bunch of services together. This isn't something to take lightly. Secondly, you've also made it possible for one service to unintentionally impact others by hogging too many resources.

## State
Instances of a stateless service don't store any information related to previous requests. An incoming request can be sent to any instance of the service. The primary benefit here is simplified operations and scaling. You can run the service behind a simple load balancer. Then, it's easy to add or remove instances as request volumes change. It's also easy to replace a failing instance.

That said, many of your services do need to store data of some kind. This data should be pushed into external services like disk bound database servers or memory bound caches.

## Documentation
A service (and its API) is only as good as its documentation. It's critical to have clear and easy-to-approach usage documentation for each service. Ideally, usage documentation for all services should be in a common place. Service teams shouldn't have to think too hard about where documentation is for a service they're using.  

Notification about changes to documented endpoints need to go out to owners of other dependent services. The notification system should have knowledge of who the current owners are, accounting for any team or ownership changes. This is information that can be tracked and made available by the platform.

## Maintaining Distributed Consistency
No matter how you look at it, consistency is hard in a distributed system. Rather than fight it, a better approach to take for a distributed system is eventual consistency. In an eventually consistent system, although services may have a divergent view of the data at any point in time, they'll eventually converge to having a consistent view.

If you've modelled your services well (i.e. loose coupling & high cohesion), you'll find that eventual consistency is a good enough default for most use cases. Creating eventually consistent distributed systems is also in line with creating loosely coupled systems. They tend to communicate asynchronously and inherently shield themselves from failures of downstream services.

Rather than fight an uphill battle, you could just have a best effort synchronization solution combined with a process which identifies and corrects inconsistencies after the fact. This approach is still eventually consistent. It's just that the window of inconsistency may be a little longer than if you had taken on the complexity of guaranteeing cross system (database & event stream) consistency.

#### Every piece of data should have a single source of truth

Even if you have to duplicate some data across multiple services, one service should always be the source of truth for any piece of data. All updates should go through the source of truth. This also becomes the originating source against which consistency verification can be done in the future.

#### Strong consistency

First, double check that you've got the service boundaries right. When services need to be strongly consistent, it usually also makes sense to colocate the data into a single service (and a single database), making it simpler to provide transactional guarantees.

If you're sure you have the right service boundaries but still need strong consistency, then you'll need to look at distributed transactions, which are difficult to implement correctly and would also strongly couple the two services together. This should be your last resort.

## Asynchronous Workers
As you embrace eventual consistency, you'll find that not everything needs to be done while the request is blocked waiting for a response. Anything that can wait (and is resource or time intensive) should be passed as jobs to asynchronous workers.

This approach:
1. **Speeds up the primary request path**. Since you're only doing a portion of the total work that needs to get done as part of the request.
2. **Spreads loads to easy to scale worker processes**. Perfect for an auto-scaling setup, where the number of workers changes dynamically based on available work to be done.
3. **Reduces error scenarios on the primary service API**. When jobs running in async workers fail, they can be retried behind the scenes without forcing the requesting service to wait.

####  Idempotency
The challenge with automatically retrying jobs is that you may not know if the failing job completed its work before it failed or not. To keep things operationally simple, you really want your jobs to be be idempotent. For our context, This means that there should be no negative impact of a job running more than once. The end result should be the same whether the job ran once or more than once.

## Shared libraries
The biggest challenge with shared libraries is that you have little control of when updates will get deployed across the services that use them. It could take days or weeks before other teams deploy the updated library. In an environment where services are independently developed and deployed, any change that requires all services to be simultaneously updated is just not practical.

The best you can do is post a deprecation schedule and coordinate with the service teams to ensure the updates get applied in a timely manner. As a result, any changes to shared libraries also need to be backwards compatible.

If it's not already obvious: shared libraries are ideal for managing auxiliary concerns like connectivity, transport, logging and monitoring. Service specific business logic should also stay out of shared libraries.

## Service templates
In addition to their core business logic, services need to manage a number of additional supplementary tasks. Some examples include: service registration, monitoring, client side load balancing, limit management and circuit breaking. Teams should be provided with templates which can be used to quickly bootstrap services which handle all these common tasks and integrate well into your platform.  

The templates should exist to speed teams up and not to enforce structure. However, *certain behaviours should be required*, like those to enable registration, monitoring and logging. Leave it to the team to decide if building something from scratch that meets behavioural requirements makes more sense than using the ready made template.

## Communication protocols
As you build more services, it becomes critical to have standardised methods of communication between them. Since services don't all have to be written in the same language, the chosen communication protocols must be language and platform independent.

#### Synchronous communication
HTTP is a great choice for synchronous communications. HTTP clients are already available in all languages. HTTP load balancers are built into cloud platforms. The protocol has built in mechanisms for caching, persistent connections, compression, authentication and encryption. Most importantly, there's an ecosystem of robust and mature tools that can be leveraged: caching servers, load balancers, excellent browser based debuggers and even proxies that can replay requests.

#### Asynchronous communication
There are two major approaches here:

* **Use a message broker**   All services push events to the broker. Other services subscribe to the events they need. In this scenario, the message broker defines its own transport protocol. Since a centralized broker can easily become a single point of failure, it's important to ensure such a system is fault tolerant and horizontally scalable.  
* **Use webhooks delivered by the services** A service exposes an endpoint from which other services can subscribe to events. It delivers those events as webhooks (i.e. an HTTP POST with a serialized message in the body) to a target destination provided at time of subscription. Such webhook deliveries should be sent by asynchronous workers that the service manages. This approach avoids introducing a single point of failure and is inherently horizontally scalable. This functionality can be built right into a service template.

## Serialisation format

**JSON** is a stable and widely used serialization format. It's natively parsed in browsers. Debuggers built into the browsers also display it well. Nothing other than a JSON parser/serializer is required, which are readily available in all languages. The main negative about using JSON is that the attribute names get repeated in every message, resulting in an inefficient use of the transport. Compression on the transport protocol can significantly mitigate this.

**Protocol buffers** are efficient to parse, efficient over the wire and heavily battle tested at Google. However, they do require language specific parser/serializer generators based on a message definition files. Language support isn't as wide as JSON, though most modern languages are covered. Servers must also share the message definition files with clients in advance.

JSON is easier to get started with and more universal. Protocol buffers keep things leaner and faster but come with a little additional development overhead in sharing and compiling .proto files. Both are good options. Pick one and use it consistently.
