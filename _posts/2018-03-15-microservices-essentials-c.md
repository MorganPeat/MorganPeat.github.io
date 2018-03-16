---
layout: post
title:  "Microservices essentials part 3 - the service"
date:   2018-03-15 13:00:00
categories: [Microservices]
tags: [microservices]
draft: false
---

There are a whole host of principles and best practices to follow when moving to microservices but I'm only going to document the main ones which will affect me and the projects I work on. Most of this is copy n' paste from a very good blog article, [Best Practices for Building a Microservice Architecture](http://www.vinaysahni.com/best-practices-for-building-a-microservice-architecture), but re-ordered it to suit my aims.  

* [Part 1 - the platform]({% post_url 2018-03-15-microservices-essentials-a %}) introduces the microservice platform, development set-up and how to identify service boundaries.
* [Part 2 - platform design]({% post_url 2018-03-15-microservices-essentials-b %}) covers the design of the overall platform (data, state, consistency, communication and serialisation protocols)
* [Part 3 - the service]({% post_url 2018-03-15-microservices-essentials-c %}) (this page) covers technical aspects to be considered for an individual service (API versioning, retry and failure policies, feature flags)

## API versioning
Versioning your API and supporting multiple versions simultaneously goes a long way to minimizing impact to other service teams. This gives them time to update their code on their own schedule. Every API should be versioned!

That said, maintaining old versions indefinitely can be challenging. Old versions should be supported for a few weeks to a few months at most, whatever is reasonable for your organization. This gives other teams the time they need without further impacting your own development speed.  

All supported versions should co-exist in the same codebase and the same service instances. Use a versioning scheme to identify which version a request is for. When possible, old endpoints should be updated to relay modified requests to the equivalent new endpoints. Although having multiple versions co-exist in the same service doesn't eliminate the complexity, it avoids accidental complexity and tackles the inherent complexity head on.

## Tolerate Unrelated Changes
Service APIs will evolve over time. A change that requires coordination with API consumers is slower to release than one that requires no coordination. To minimize coupling, services should be able to tolerate unrelated changes to responses of services they communicate with. This just means they shouldn't break if a field is added or an unused field is changed/removed.

If all services tolerated unrelated changes, additive API changes could be made without any coordination. Unrelated breaking changes would just require consuming service teams to run through their test suite to verify everything is still working.

## Short timeouts
Imagine this scenario: One service gets overloaded with requests and slows down. As a result, all services calling it slow down. Eventually the user interfaces are lagging. This is a cascading failure: many services failing and raising alerts at the same time.  

With multiple services backing up and failing, identifying the source of the problem becomes a challenge. Is a service having an internal problem or is it a result of a downstream service? Using short timeouts on downstream API calls can help in this scenario. Timeouts prevent multiple services from just slowing down. Instead, you'll have one service really failing and other services failing fast and pointing to it.

It's not good enough to just have a default 30 second timeout. You need a timeout that tightly covers what's reasonably expected of a downstream service. For example, if you expect a service to respond within 10 - 50 milliseconds, than any timeout over 500 milliseconds is already more than enough.

## Circuit breakers
Every attempt to communicate with a failing resource has a cost. It uses resources on the consumer side to try to make a request, it uses up network resources and it uses up resources on the target side.

A circuit breaker prevents requests that are doomed to fail from even being attempted. Implementing this is straight forward: if requests to a service are resulting in a high number of failures, flip a flag and stop trying to send requests to service for a period of time. Periodically allow a single request through to see if the service is back online, so you can flip the flag back.

## Automatic retry
When you're failing fast, it makes sense to automatically retry certain kinds of requests. This is especially the case for asynchronous communications.

A service that's down can easily get hammered when it comes back online if a bunch of other services were retrying at the same retry window. This is also known as a [thundering herd](https://en.wikipedia.org/wiki/Thundering_herd_problem), which can be easily avoided this by using randomized retry windows. If your infrastructure doesn't implement circuit breakers, I recommend combining randomized retry windows with an exponential backoff to further spread out requests.

## Correlation IDs
A single user request can result in activity occurring across many services, which makes things difficult when trying to debug the impact of a specific request. One way to make things simpler is to include a correlation ID in service requests. A correlation ID is a unique identifier for the originating request that is passed by each service to any downstream requests. When combined with a centralized logging layer, this makes it really easy to see a request make its way through your infrastructure.

The IDs are generated by either user facing aggregation service or by any service that needs to make a request that's not an immediate side effect of an incoming request. Any sufficiently random string (like a UUID) would do the trick.

## Feature flags
A feature flag is code that lets you turn on or off specific features at run time, effectively decoupling code deployment from feature deployment. This enables you to *deploy* the code for a feature incrementally over a period of time. Then, you can *release* the feature to the users when you're ready.

You will need an interface to view and manage feature flags on the platform. The code to lookup the flags can be included in a shared library.

#### Incremental releases
Feature flags make it possible to release features to sets of users in phases. Perhaps to 10% of your users at first or perhaps only to users in a specific region. The advantage here is that you'll have an opportunity to identify problems before they impact a large percentage of your users. It also enables quick rollback of the feature by turning off the flag.

#### Short-lived
Feature flags should exist only until the feature is successfully deployed. Long running flags are a bad idea: they make it harder to support the users (as they'll be experiencing different behaviors), harder to test the system (with many code paths) and harder to debug the system. A flag should be scheduled to be removed soon after the feature is fully deployed.

#### At the entry point
The point of a feature flag is to decouple feature deployment from code deployment. For this, you just need to wrap the flag around the entry point to the feature, not all the code paths related to it. As an example, for a user interface visible feature, a flag could just hide the link/button in the interface to get to the feature.

## Authentication
All API requests should be authenticated. This helps service teams better analyse usage patterns and provides an identifier which can be used to manage consumer specific limits.

The identifiers would be unique API keys provided by the service team to consumers who use the service. You'll need some way to issue and revoke these API keys. This could either be built into the service template or be provided as a centralized authentication service at the platform level to enable self-service management of keys by service teams.
