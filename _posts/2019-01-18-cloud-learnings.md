---
layout: post
title:  "Cloud Learnings - results of a pilot cloud project"
date:   2019-01-18 12:00:00
categories: [CloudMigration]
tags: [cloud, cloud-plan]
draft: false
---

I recently worked on a 3-month pilot project to investigate how we can move my firm's applications into the cloud. For my part, I concentrated on deploying Windows .NET services and ASP.NET applications.

# tl;dr

* Application code doesn't need to change too much, but the way it is deployed does.
* In the locked-down corporate environment, not much cloud-specific training is needed.
* A working knowledge of Infrastructure as Code and Configuration as Code is needed. For those who will be reviewing pull requests containing IaC / CaC changes, a deeper knowledge is needed.
* Even when IaC / CaC is used having a "sandpit" (where one can play with cloud tools manually) is important.

# Priorities

A few headline things that need to be done.

## Must do

* Encrypt everything (in transit, at rest)
* Develop common deployment scripts (IIS, services, PKI, config) the for CI/CD pipeline
* Replace applications' deployment frameworks with a common Cloud CI/CD pipeline
* Ensure no secrets (connection strings, passwords, confidential info) are logged
* Rewrite any app-level monitoring to the new platform

## Should do

* Log in json format, move to semantic logging
* Integrate application metrics libraries
* Move to better secrets model (on demand from Vault)
* Improve DevOps (CI, CD, immutable infrastructure)

## Could do

* Share more (nuget libraries for cross-cutting concerns, CI/CD scripts)
* Move to .NET Core (push for internal depedencies to be moved, too)
* Move to containers (H2 2019, when Windows container support is available)

# Application-level specifics

It shouldn't be hard to lift n' shift apps onto the cloud. The changes I had to make to my app were minimal and when we actually get something up and running I don't expect much more will need to change. The main changes / considerations are:

## Logging

Most logging systems (Stackdriver, DataDog) integrate with log4net or recommend an installed agent tailing a log file. All solutions suggest logging in json format but can parse strings, allowing a slower pace of conversion to semantic logging. My firm wish to have a single centralised logging solution so a piece of work may need to be done to ensure no secrets are logged.

## Monitoring

OS-level metrics are captured by default by all cloud providers and will be fed into the firm's monitoring solution.  
Libraries for application metrics are available for most monitoring systems (e.g. Stackdriver, DataDog) but will need to be integrated.  

## Configuration

I think all applications where I work use some form of templating to build config files so adding a new "cloud" environment should be straightforward. In many of our apps configuration is coupled to the package (e.g. all environment configuration files are embedded within a MSI and selected via a parameter) but in cloud it must be supplied at deployment time: some secrets (e.g. DB password, TLS certificate) cannot be determined during package build. So work will need to be done to render configuration files at deploy-time instead of build-time.

## TLS

This will probably be mandatory in cloud. I have proved the process using Hashicorp Vault's PKI plugin and it works smoothly. App teams can be largely ignorant of TLS beyond knowing which app-specific parameters (CN, SAN) to pass. IIS bindings for HTTPS should be done by the CI/CD framework; most DBs have TLS-aware drivers which need little more than a change of connection string.

## Packaging & deployment

For the pilot I packaged my app as a simple zip of binaries, it was good enough. The CI/CD pipeline can take care of any deployment steps and through the use of common modules will enable re-use of deployment scripts between apps. I suspect it will be easier to re-package applications to use a cloud CI/CD solution (e.g. Chef, Ansible) rather than retro-fit cloud-specific steps (configuring secrets, TLS, logging) to each application. Once a pattern is established it should be straightforward and will help to remove the myriad deployment frameworks in use.

## Cloud Native (containers)

A gradual move to .NET Core is underway in my firm but is mostly dependent on internal third-party libraries which haven't thought about moving yet. Otherwise containerisation will have to wait until H2 2019 when full-fat .NET Framework is supported on Windows.

# The cloud platform

Other than application-level changes we need to consider how the wider cloud operating model will affect our application.  

## Training

Some basic training is needed in order to understand how cloud computing works but I think it unnecessary to delve deep. Any reasonably controlling corporate platform will abstract many cloud things like VPC and IAM; cloud native tools (DBs, messaging, GKE) are probably out of scope during the initial migration phase while applications concentrate on migrating.

## Principal of Least Privilege

This is a great thing to have, to restrict access to everything except things you really need, but it needs to be well thought out.

* RDP - Cloud shell works great for Linux boxes but generally the Windows story is woeful for any corporate environment (that won't allow port 3389). Hopefully my firm will permit RDP otherwise we may need to trial a RDP-over-HTTPS solution.
* CI/CD projects and source code repositories - By default (in my firm) a user can only see their own projects, there is no read-only default access to other projects. A open model (where everything is visible read only) would encourage better code reuse and sharing of best practice ("given enough eyeballs, all bugs are shallow"). This should include, where possible, the platform and infrastructure projects so we can all see how the a good corporate platform is designed.
* Networking - It would be more secure (although greater burden) if each application team's IaC could control a custom network and have control over ingress and egress.
* Bastion hosts and VM access - I think this is a great thing to have and ultimately it would be good if we can avoid logging on to VMs at all. But more work around DevOps processes is needed where I am before we can reach that level of technical maturity.

## Images

My firm enforces custom ("hardened") base images but the process is immature. We have used images that don't work properly, images that are subsequently blacklisted (bringing down the entire cloud project) and are not notified when new images are available or what is in them. A suitable process is needed: a catalogue of images with complete description with alerts when changes are made.

## Billing

As we use IaC (Terraform) and have no permission to manually create VMs we use a billing dashboard to give a detailed breakdown of cost. It has a 24-hour lag which is still way better than we ever had before but it would be good to get quicker feedback on costs during development. Perhaps a sandpit environment would provide this as all cloud providers shows VM costs on creation.

# Deployment pipeline

## Infrastructure as Code

Terraform is an essential skill to have and, as there is quite a learning curve, it may be worth looking into some training. Most people may get away with copy n' paste and minor changes but it's worth a few people getting skilled.  
Knowledge of the specific cloud provider's products and general cloud concepts is needed.  
Terraform tooling is good (I use VS Code which gives intellisense, the TF CLI lints and validates).  
As IaC is all version controlled code review will prevent unwanted change. TF Enterprise can help by enforcing policies but otherwise there will need to be some experts that can understand and validate changes.

## Configuration as Code

There are almost as many deployment frameworks as there are apps in my firm so modifying each one to suit the cloud will be time consuming. The main configuration management tools (Puppet, Chef, Ansible, Salt) all have a considerable learning curve (Chef and Puppet the most) but all permit parameterised modules so a set of shared modules will suit most use cases.  
The main CaC tools generally configure Windows using Powershell scripts, Powershell DSC and Chocolatey. None of this will cause a high barrier of entry to us Windows developers!  
Some deployment steps are only possible in the context of cloud (e.g. calling Vault to request a new TLS certificate) so consolidating on a common set of reusable modules would be the quickest way to migrate to cloud.  
Startup scripts are hard to develop and test whereas CaC is much easier. Standardised bootstrap scripts that initialise the CaC pipeline are desired.

## Sandpit

Corporate environments are mostly locked down like a production environment. The need for a "sandpit" is understood by most but we need to make sure that it permits the following:

* We need to be able to manually create VMs through the cloud UI (they show billing costs, are quick for ad-hoc testing and development)
* We need to be able to use cloud CLI tools from developer workstations (e.g. for tailing logs, restarting instances)
* We need to be able to use terraform CLI from developer workstations (e.g. for developing and testing TF scripts, tainting instances)
