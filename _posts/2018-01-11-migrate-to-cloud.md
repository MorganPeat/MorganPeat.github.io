---
layout: post
title:  "Migrating an on-premises app to the cloud - 1"
date:   2018-01-11 16:00:00
categories: [CloudMigration]
tags: [cloud, cloud-plan]
draft : false
---

I have just started working on a proof of concept project to migrate a .Net application from running on in-house infrastructure to "The Cloud". It sounds like a great opportunity to learn new skills and play with some exciting new technology, but clearly it begs a whole host of questions...

## Why do we want to migrate to The Cloud?
Because it's cool, obviously! But on a more sensible note, there are several clear reasons:
### Cost savings
Cost is almost always the main driver of projects where I am! Managers look to Cloud as a way to save money. Two obvious opprtunities are:
#### Hardware
Your app may require just a single low-spec VM most of the time but you will need to have enough "grunt" hanging around to ensure that performance doesn't degrade too much under high load. I hear rumours that many companies normally run at between 5% to 20% capacity which is a lot of hardware standing idle, and a lot of wasted money!
With Cloud there is the alure of [auto-scaling](https://en.wikipedia.org/wiki/Autoscaling). Here you run under your minimum configuration (e.g. a single low-spec VM) but when your application starts to come under load it can automatically add more VMs as required to maintain performance. Once the period of heavy load has passed your application will scale back down again.
#### Infrastructure
The other main driver in Cloud projects is that the hardware and infrastructure will be hosted by a Cloud vendor so all the gubbins that is required for on-premises applications (load balancers, routers, switches, data centres, etc, etc) is not needed. Instead it becomes someone elses problem, and we lose much of the associated cost (hardware and people) and headache.

### Agility
Getting new hardware is tough in the corporate environment. If an application is coming under heavy load more frequently and an additional server is needed, or if the hardware is reaching end of life and needs an upgrade, it could take a good month or two to approve, order, install and set up. In a Cloud environment you could merely click the "add / upgrade a VM" button in the Cloud UI of your choice, make a cup of tea, then come back and start using it.

## What does it mean to "run on The Cloud"?
The Project Manager's dilemma: What does "done" look like? My initial guess: our application will continue to do whatever it did before except it will run on Cloud hardware not on our own. There are a host of questions and assumptions around this, which is what I'll be exploring over the course of this project.
#### Is it reasonable to expect _everything_ to run on Cloud kit?
There may be some constraints that make this difficult: third-party hardware or connectivity that may be difficult to get, confidential data that the company (or regulator) is unwilling to let out of the door, or vast amounts of data that could be prohibitively expensive to host anywhere else. Quite what we do here is also up in the air, and subject to more investigation. (Hybrid cloud?)

## How do we get to this end state?
Exactly how to do all this is undecided and open to discussion at the moment (hence "proof of concept") so the initial plan is pretty simple:
![project plan]({{ site.baseurl }}/images/MigrationPlanv0.1.png "Project Plan v0.1")
So far, so woolly.

## tl;dr
Cloud is cool, should save the company money, and allow us to move quicker. Currently we have no idea how to do it!
