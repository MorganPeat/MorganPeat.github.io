---
layout: post
title:  "Keeping accurate time on GCE"
date:   2019-02-01 16:00:00
categories: [CloudMigration]
tags: [cloud]
draft: false
excerpt_separator: <!--more-->
---

[A year ago]({% post_url 2018-01-31-time-sync-on-cloud-vms %}) I looked at how to keep VM system clocks accurate to UTC. This post details another experiment, this time using just Google Compute Engine VMs.

This time, the experiment aims to prove that Virtual Machines running on Google Compute Engine (i.e. IaaS) can keep the system clock accurate to within 1ms of UTC. I ran a test against each of the operating systems that my company will be using most: RHEL 7 and Windows 2016.

<!--more-->

# Reference clock

Google maintains a [public NTP service](https://developers.google.com/time/). GCE instances can use (and indeed are pre-configured to point to) `metadata.google.internal`, the internal address of their NTP services. Although there is [no SLA](https://developers.google.com/time/faq#sla) for the service it appears to be accurate and [has been called](https://www.fsmlabs.com/news/2015/06/30/cloud.html) "the best public time source" by a company that manufactures custom clock-sync hardware and software.

# RHEL 7

The GCE RHEL-7 images configure [chrony](https://chrony.tuxfamily.org/) by default for clock sync and point it to Google's internal NTP service. I used a simple Terraform script to spin up my VM and tell chrony to log some measurements which can be used to measure accuracy.

{% highlight hcl %}
resource "google_compute_instance" "rhel-vm" {
  name         = "rhel7-clock-sync-test"
  machine_type = "f1-micro"
  zone         = "europe-west2-b"

  boot_disk {
    initialize_params {
      image = "rhel-7-v20180516"
    }
  }

  network_interface {
    network = "default"

    access_config {
      // Include this section to give the VM an external ip address
    }
  }

  metadata_startup_script = "${file("${path.module}/scripts/bootstrap_rhel.sh")}"
}
{% endhighlight %}

{% highlight bash %}
#!/bin/bash

cat > /etc/chrony.conf << EOL
server metadata.google.internal iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
log measurements statistics tracking
EOL

systemctl restart chronyd
{% endhighlight %}

After leaving the VM for a few  hours I looked at the chrony logs. It seems that the best measure of system clock accuracy comes from the `Offset` column in `/var/log/chrony/measurements.log`:

The offset column [contains](https://chrony.tuxfamily.org/doc/3.2/chrony.conf.html#log):
> The estimated local clock error (theta in RFC 5905)
..and [RFC 5905](https://tools.ietf.org/html/rfc5905) says :
> The offset (theta) represents the maximum-likelihood time offset of the server clock relative to the system clock.

![Offset]({{ site.baseurl }}/images/clock-sync/rhel7.png "Offset")

This image shows that, after an initial period where the clock stabilised itself, the VM system clock stayed accurate to ~0.2ms.

# Windows 2016

Windows 2016 server contains [many improvements](https://docs.microsoft.com/en-us/windows-server/networking/windows-time-service/accurate-time) that give a more accurate system clock with NTP, and make it more accurate when running in a VM.  
Again, the GCE VM comes pre-configured to point to Google's internal NTP server but, just to make sure, I set it up in my Terraform startup script:

{% highlight hcl %}
resource "google_compute_instance" "win-vm" {
  name         = "win2016-clock-sync-test"
  machine_type = "n1-standard-1"
  zone         = "europe-west2-b"

  boot_disk {
    initialize_params {
      image = "windows-server-2016-dc-v20190108"
    }
  }

  network_interface {
    network = "default"

    access_config {
      // Include this section to give the VM an external ip address
    }
  }

  metadata_startup_script = "w32tm /config /manualpeerlist:metadata.google.internal /syncfromflags:manual /update"
}
{% endhighlight %}

For Windows I didn't set up any logging for the time service, although there have been some [improvements in traceability](https://docs.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-for-traceability) here also.  
So once the VM had started up I logged in using RDP and ran

```
w32tm /stripchart /computer:metadata.google.internal /dataonly /period:120 > c:\time.log
```

...and downloaded the log file after a few hours.

![Offset]({{ site.baseurl }}/images/clock-sync/win2k16.png "Offset")

This image shows that, after an initial period where the clock stabilised itself, the VM system clock stayed accurate to ~0.4ms. This is not quite as good as RHEL but is easily within the 1ms tolerance that I'd need for MiFID II.
