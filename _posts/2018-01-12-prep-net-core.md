---
layout: post
title:  "Migrating to the cloud - 2 - prep work - .Net Core"
date:   2018-01-12 12:00:00
categories: [CloudMigration]
tags: [cloud, cloud-prep, netcoreapp]
---

Without any further investigation into what is needed to run .Net applications on the Cloud it is clear that some preparatory work can take place now. One such piece of work is to migrate application code from targeting the .Net Framework to targeting .Net Core.

## Why bother?
Cost! .Net Core code can run cross-platform which means it can be run on Linux, not just on Windows. A back-of-a-fag-packet look at Amazon's EC2 on-demand pricing shows a "Linux" instance (t2.large in London) costing about 10 cents / hour versus a similar "Windows" instance costing 13 cents. The difference is presumably due to licensing costs and is enough to make us want to consider re-targeting our code.

## How to do it?
There are plenty of articles around which describe the methods, but the general advice seems to be:
* First make sure all Nuget dependencies can target .Net Core too.
* Don't bother trying to convert a csproj file; just start from scratch and re-add dependencies as needed.

The blog articles I found most helpful are these two:
* [How to port from .net framework to .net standard](https://codehollow.com/2017/05/port-net-framework-net-standard/)
* [Old csproj to new csproj: Visual Studio 2017 upgrade guide](http://www.natemcmaster.com/blog/2017/03/09/vs2015-to-vs2017-upgrade/)

## Caveats
So far we have just been moving our in-house Nuget libraries to .Net Core and haven't yet looked at executables or ASP.Net Core. That will be the subject of further investigation.

## tl;dr
By targeting .Net Core you can run your Cloud application on Linux which is cheaper than Windows.
