---
layout: post
title:  "On time - keeping an accurate clock on Cloud VMs"
date:   2018-01-31 10:00:00
categories: [CloudMigration]
Xtags: [cloud]
tags: [draft]
draft : true
---
Some systems where I work come under the MiFID II requirements for clock synchronisation, known as [RTS-25](http://eur-lex.europa.eu/legal-content/EN/TXT/?uri=uriserv:OJ.L_.2017.087.01.0148.01.ENG&toc=OJ:L:2017:087:TOC). These guidelines state that
> Operators of trading venues and their members or participants shall synchronise the business clocks they use to record the date and time of any reportable event. ... Operators of trading venues shall ensure that their business clocks adhere to the levels of accuracy specified in Table 1.

Practically, for some systems I am am involved with, this means the system clock must be within 1s of UTC. For others it must be within 1ms.  
How to achieve this on Cloud VMs?  

## NTP Servers
From my experiments with Windows VMs it seems that Azure VMs are set up to sync to `time.windows.com` by default. Google Compute VMs sync to `metadata.google.internal` but AWS VMs are left as default.

### Amazon Time Sync Service
A relatively new (November 2017) service on AWS should help to keep clocks in sync. It ...
> utilizes a fleet of redundant satellite-connected and atomic reference clocks in AWS regions to deliver current time readings of the Coordinated Universal Time (UTC) global standard. The service is designed to be highly available with a continuously monitored time infrastructure and provides a low variance reference time source

Basically this means providing a well-known IP address (169.254.169.123) in all AWS regions to which you can point your NTP time-keeping service.
* [Introducing the Amazon Time Sync Service](https://aws.amazon.com/about-aws/whats-new/2017/11/introducing-the-amazon-time-sync-service/)
* [Keeping Time With Amazon Time Sync Service](https://aws.amazon.com/blogs/aws/keeping-time-with-amazon-time-sync-service/)
* [Setting the Time for a Windows Instance](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/windows-set-time.html)
* [Setting the Time for Your Linux Instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html)

### Google Public NTP
Google host NTP infrastructure in several of their datacentres. GCP VMs are pre-configured to use `metadata.google.internal`. Other users may use their public NTP service at `time.google.com`
* [Google Public NTP](https://developers.google.com/time/)
* [Set up network time protocol (NTP) for instances](https://cloud.google.com/compute/docs/instances/managing-instances#configure-ntp)

### Azure
I can find surprisingly little about clock sync on Azure VMs. It certainly seems to be set up by default to sync against `time.windows.com`, and that name should be
> resolved by your ISP, which should point to a Microsoft owned resource

So my guess is that Azure VMs are pre-configured to point to some reliable NTP infrastructure. All I can find is a few links:
* [Windows Server 2016 Accurate Time](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/windows-time-service/accurate-time)
* [How to validate local VM clock with NTP on Windows Azure?](https://stackoverflow.com/questions/6186776/how-to-validate-local-vm-clock-with-ntp-on-windows-azure)
* [Azure FAQ: How frequently is the clock on my Windows Azure VM synchronized?](https://blog.codingoutloud.com/2011/08/25/azure-faq-how-frequently-is-the-clock-on-my-windows-azure-vm-synchronized/)

<br/>
<br/>

## Leap seconds
From [wikipedia](https://en.wikipedia.org/wiki/Leap_second):
> A leap second is a one-second adjustment that is occasionally applied to Coordinated Universal Time (UTC) in order to keep its time of day close to the mean solar time. <br/>
> The duration of one mean solar day is now slightly longer than 24 hours because the rotation of the Earth has slowed down. Therefore ... the UTC time-of-day would slowly drift apart from that of solar-based standards, such as Greenwich Mean Time (GMT). <br/>
The purpose of a leap second is to compensate for this drift, by occasionally scheduling some UTC days with [an extra second].

It seems (although I can't find a source for this) that the MiFID II approach is to 'jump' when a leap second is applied; that means either that a minute will have an extra second (i.e. 58, 59, 60 or 58, 59, 59). This may cause a problem for systems that generate unique ids by using the clock as for a short period of time, the clock may not increase.

### Smearing time
[Amazon's approach](https://aws.amazon.com/blogs/aws/look-before-you-leap-the-coming-leap-second-and-aws/) to leap seconds is to smooth out the additional second over a 24-hour period, 12 hours each side:
> The Amazon Time Sync Service automatically smooths out (smears) leap seconds that are periodically added to UTC, so that customers do not have to worry about application errors due to their addition.

[Google's approach](https://cloudplatform.googleblog.com/2016/11/making-every-leap-second-count-with-our-new-public-NTP-servers.html) is to smooth out over 10 hours each side:
>  Instead of adding a single extra second to the end of the day, we'll run the clocks 0.0014% slower across the ten hours before and ten hours after the leap second, and “smear” the extra second across these twenty hours.

The upshot of this is that our Cloud VM clocks may tell a different time depending on their NTP server, regardless of how accurately the clock thinks it is tracking UTC. This means Cloud VMs may not meet MiDID II requirements, at least on those irregular days where a leap second may take place.

However, Amazon says:
> In the future, we will also provide mechanisms for accessing non-leap smeared time.

This sort of service may provide the answer, although any services using that clock would need to be able to cope with non-increasing time during leap seconds.

<br/>
<br/>

## Quick and dirty test
I ran a quick n' dirty test to compare Cloud VM clocks across three Cloud vendors. My approach was as follows:
1. Run the same test on VMs in AWS, GCP, Azure
2. Run a Windows test against Windows 2016 Datacentre OS
3. Run a Linux test against RHEL 7 (see note on Azure, however)
4. On each VM, sync the clock against the recommended NTP server
5. Over a period of time, compare the clock against the NTP

### Windows
For the AWS VM I configured it as per [these instructions](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/windows-set-time.html). GCP and Azure appeared to be pre-configured. I ran the command
{% highlight shell %}
W32tm /stripchart /dataonly /computer:<NTP server> /period:30
{% endhighlight %}
... to compare the system clock against a NTP server. For some attempt at consistency I compared against `time.windows.com` and `time.nist.gov` for all tests. The tests ran for varying lengths of time (at least half an hour per VM) and a sample was taken every 30 seconds. (Generally there are fewer plots against `time.nist.gov` due to timeouts).

#### AWS
![Windows 2016 in AWS]({{ site.baseurl }}/images/cloud-poc/aws-windows.png "Windows 2016 in AWS")
#### GCP
![Windows 2016 in GCP]({{ site.baseurl }}/images/cloud-poc/gcp-windows.png "Windows 2016 in GCP")
#### Azure
![Windows 2016 in Azure]({{ site.baseurl }}/images/cloud-poc/msa-windows.png "Windows 2016 in Azure")

There isn't much that can be gleaned from this! A few tentative conclusions (i.e. wild guesses) would be:
* You can see the local clock drift back into line periodically. I assume this happens when a sync is performed vs the NTP server
* All the VMs managed to stay well within a second of UTC
* Windows seemed to be the closest to 'true' UTC
* All VMs would drift out of line quite quickly, and needed a re-sync to come back into line

<br/>

### Linux
I installed `chrony` on all Linux boxes as per the [AWS instructions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-time.html), using the most appropriate NTP server as the 'preferred' source. Again I compared against `time.windows.com` and `time.nist.gov` for all tests.  
I ran the command
{% highlight shell %}
watch -n 30 "chronyc sourcestats | tee -a logfile"
{% endhighlight %}
... to log the offset vs the NTP server every 30 seconds. I created a RHEL 7 VM in AWS and GCP but my Azure account (MSDN Enterprise) won't let me create RHEL 7 so I used the [closest thing](http://www.sharkyforums.com/showthread.php?308033-What-s-the-closest-free-distro-to-Red-Hat-Linux) I could find, CentOS 6.8.

#### AWS
![RHEL 7 in AWS]({{ site.baseurl }}/images/cloud-poc/aws-windows.png "RHEL 7 in AWS")
#### GCP
![RHEL 7 in GCP]({{ site.baseurl }}/images/cloud-poc/gcp-windows.png "RHEL 7 in GCP")
#### Azure
![CentOS 6.8 in Azure]({{ site.baseurl }}/images/cloud-poc/msa-windows.png "CentOS 6.8 in Azure")

Again, possibly not much to be gleaned from this. I'm not even sure my methodology is sound! (Hence calling it a "quick and dirty" test). But on first glance (i.e. more wild guesswork) it seem:
* Linux (particularly with `chrony`) kicks Windows' ass, even with 2016 server which is supposed to have some clock accuracy improvements
* All local clocks stayed within 0.1ms of UTC when compared to their NTP server

Looks like, at first glance, we'll be trying to run all our clock-sensitive .NET Core services on Linux!
