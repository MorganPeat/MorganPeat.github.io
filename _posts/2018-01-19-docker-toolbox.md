---
layout: post
title:  "Installing Docker toolbox on Windows 7"
date:   2018-01-19 15:00:00
categories: [Docker]
tags: [docker]
---
Where I work we are still on Windows 7 development machines which means we can't use Docker for Windows - [Windows 10 is required](https://docs.docker.com/docker-for-windows/faqs/#why-is-windows-10-required). Luckily there is [Docker Toolbox](https://docs.docker.com/toolbox/overview/), the "Legacy desktop solution" for systems that don't meet the minimum requirements. Here a small Linux VM is spun up which will host Docker. I spent a couple days getting it set up and configuring it to work in my company. This post details the steps and workarounds I took.

## Installing Docker Toolbox
The [Docker Toolbox on Windows](https://docs.docker.com/toolbox/toolbox_install_windows/) page provides the download link and installation instructions. The pre-requisites are:
* Your machine must have a 64-bit operating system running Windows 7 or higher
* Virtualization must be enabled on your machine (BIOS setting; I had to get my helpdesk to do this since my PC is not located anywhere near my desk)
Once installed, you fire up the Docker Quickstart Terminal.
Boom!
{% highlight shell %}
Running pre-create checks...
(default) No default Boot2Docker ISO found locally, downloading the latest release...
Error with pre-create check: "Get https://api.github.com/repos/boot2docker/boot2docker/releases/latest: dial tcp: lookup api.github.com: no such host"
Looks like something went wrong in step ´Checking if machine default exists´...
{% endhighlight %}

## Getting the Boot2Docker ISO image
I think the issue is that the quickstart terminal doesn't have access through the corporate firewall so is unable to locate the ISO. A [forum post](https://forums.docker.com/t/pre-create-check-failed-when-first-time-launch-docker-quickstart-terminal/9977/3)  gives a workaround:
* Download the latest boot2docker image from https://github.com/boot2docker/boot2docker/releases (v18.01.0-ce for me)
* Save it to the Docker local cache in `c:\\user\\USERNAME.docker\\machine\\cache`

Success!
![Docker terminal]({{ site.baseurl }}/images/docker-terminal.png "Docker terminal")

## Hello, world
Next up, run the hello world sample:
{% highlight shell %}
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
C:\Program Files\Docker Toolbox\docker.exe: Error response from daemon: Get https://registry-1.docker.io/v2/: dial tcp: lookup registry-1.docker.io on 10.0.2.3:53: no such host.
See 'C:\Program Files\Docker Toolbox\docker.exe run --help'.
{% endhighlight %}

Oh dear! It's the proxy server again. But this time its the new boot2docker VM that can't get through. Google to the rescue!
* https://stackoverflow.com/questions/24489265/docker-boot2docker-set-http-https-proxies-for-docker-on-os-x
* https://docs.docker.com/toolbox/faqs/troubleshoot/#configure-http-proxy-settings-on-docker-machines

I chose the option of editing the config within the VM, rather than supplying parameters. YMMV. What I did:
* `docker-machine ssh` to get a terminal session in the VM
* `sudo vi /var/lib/boot2docker/profile` to edit the profile settings with write access
* Add settings for `HTTP_PROXY` and `HTTPS_PROXY`: `export HTTP_PROXY=user:password@your.proxy.name:8080`
* `sudo /etc/init.d/docker restart` to restart the VM docker service and pick up the new settings
* `exit` to leave the VM terminal session

Try again:
{% highlight shell %}
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
C:\Program Files\Docker Toolbox\docker.exe: Get https://registry-1.docker.io/v2/library/hello-world/manifests/sha256:8072a54ebb3bc136150e2f2860f00a7bf45f13eeb917cca2430fcd0054c8e51b: net/http: TLS handshake timeout.
See 'C:\Program Files\Docker Toolbox\docker.exe run --help'.
{% endhighlight %}
