---
layout: post
title:  "Microservice spike part 4 - run in an Istio service mesh"
date:   2018-08-30 12:00:00
categories: [Microservices]
tags: [cloud, k8s]
draft: false
---

This is part 4 in a series of posts where I spike out a cloud microservice app on GCP. In this post I set up an [Istio](https://istio.io/) service mesh to run my app.

* [Part 1 - the CryptoTracker app]({% post_url 2018-08-21-cryptotracker-a %}) introduces my prototype microservice app
* [Part 2 - run in GCP]({% post_url 2018-08-21-cryptotracker-b %}) gets the app running in Docker in Google Compute Engine
* [Part 3 - run in GKE]({% post_url 2018-08-23-cryptotracker-c %}) gets the app running in Google Kubernetes Engine
* [Part 4 - run in an Istio service mesh]({% post_url 2018-08-30-cryptotracker-d %}) (this page) gets the app running in an Istio service mesh


## Install Istio

There are [many](https://github.com/istio/istio/tree/master/install/gcp/deployment_manager) [articles](https://istio.io/docs/setup/kubernetes/quick-start-gke-dm/) which describe how to install [Istio](https://istio.io/) but they all seem to add the [Bookinfo](https://istio.io/docs/examples/bookinfo/) sample application, even if you deselect it! Given that I'm paying for GCP (well, I have a limited budget with my free trial) I want to keep my usage as small as possible.  

Since I want to play with [Helm](https://helm.sh/) package manager anyway, I decided to [install with Helm via `helm template`](https://istio.io/docs/setup/kubernetes/helm-install/#option-1-install-with-helm-via-helm-template).


### Install helm client

Pretty straightforward instructions on the [Helm github site](https://github.com/helm/helm/blob/master/docs/install.md):
* In the GCP Cloud Shell I downloaded the Helm package using `wget https://storage.googleapis.com/kubernetes-helm/helm-v2.10.0-linux-amd64.tar.gz`
* Following the instructions I extracted and moved the `helm` client binary to my `$HOME` folder so it's accessible on my path

### Download and set up Istio

Instructions on the [Istio site](https://istio.io/docs/setup/kubernetes/download-release/), but I simply ran `curl -L https://git.io/getLatestIstio | sh -` which downloaded and unpacked the latest Istio release (1.0.1 for me). It even gives the `export PATH` command needed to add `istioctl` to the path.  

Istio's [GKE-specific](https://istio.io/docs/setup/kubernetes/platform-setup/gke/) instructions detail how to get my k8s cluster ready:
* I set my Cloud console up to talk to k8s using `gcloud container clusters get-credentials ctmd-cluster --zone europe-west1-b`
* I gave my account cluster admin permission using `kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)`
* I then created a namespace which Istio will run under: `kubectl create namespace istio-system`  


Finally, I can install Istio. First I used the `helm` client to create my k8s manifest. From `$HOME` I ran `./helm template istio-1.0.1/install/kubernetes/helm/istio --name istio --namespace istio-system > $HOME/istio.yaml`.  This created a ~120Kb `istio.yaml` file which I can then install into my cluster with `kubectl apply -f $HOME/istio.yaml`.

A whole bunch of Istio-related stuff was installed into my cluster (but _not_ the Bookinfo sample app!) and, after a while, Istio is up and running!

![Istio running in my GKE cluster]({{ site.baseurl }}/images/cryptotracker/istio-1.png "Istio running in my GKE cluster")



## Install my app into Istio

Next thing is to get my application up and running. Rather than use the GKE UI I'm using yaml manifest files, these will be used by Istio to inject its [sidecar proxies](https://istio.io/docs/concepts/what-is-istio/#envoy).


### k8s manifest

In my [previous post]({% post_url 2018-08-23-cryptotracker-c %}) I used the GKE UI to deploy my app. Once running I can view the generated yaml manifest for the deployment and service and use this to create a k8s manifest. I can then deploy this manifest into my Istio-controlled cluster via `kubectl apply -f <(istioctl kube-inject -f ctmd.yaml)`. This used `istioctl` to inject the sidecars and other Istio config into my manifest, then run the whole lot into my cluster via `kubectl`.

My app is up and running in Istio! It crashes, of course, but that's because I'm missing a few bits of config.


### Mongo connection string

My app expects the environment variable `MongoDB__ConnectionString` to be present. While I could set this via the GKE UI before, now it needs to be set via the command line / a manifest. Since the environment variable represents a connection string and contains the username and password needed to connect to Atlas, it should be considered a "secret". I decided to configure my app manifest to look for the environment variable from a [kubernetes secret](https://kubernetes.io/docs/concepts/configuration/secret/). Rather than permit the secret to be stored in a manifest or source control, I set it manually in the Cloud shell using `kubectl create secret generic ctmd-config --from-literal=MongoDB__ConnectionString=<my-mongo-connection-string>`. I can then instruct my deployment to source this environment variable from my secret:

{% highlight shell %}
env:
- name: MongoDB__ConnectionString
  valueFrom:
    secretKeyRef:
      key: MongoDB__ConnectionString
      name: ctmd-config
{% endhighlight %}


### Egress to MongoDB Atlas

From the [Istio egress docs](https://istio.io/docs/tasks/traffic-management/egress/):

> By default, Istio-enabled services are unable to access URLs outside of the cluster because the pod uses iptables to transparently redirect all outbound traffic to the sidecar proxy, which only handles intra-cluster destinations.

I created an istio `ServiceEntry` to permit egress to MongoDB Atlas:

{% highlight shell %}
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: mongo-ext
spec:
  hosts:
  - ignored-as-not-http.com
  addresses:
  - 0.0.0.0/0
  ports:
  - number: 27017
    name: mongo
    protocol: MONGO
  location: MESH_EXTERNAL
  resolution: NONE  
{% endhighlight %}

The reference docs for `ServiceEntry` are [here](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry). More details, and a good explanation about `ServiceEntry`, are available [here](https://istio.io/blog/2018/v1alpha3-routing/). I used both those links to create the yaml.

* `hosts`: This is ignored for non-http protocols
* `addresses`: To keep things simple (and because I don't want to hardcode my Atlas IP addresses) I permit access to any IP address
* `ports`:  Egress is permitted to the default mongo port
* `location`: Indicates that this service is external to the mesh. See [ServiceEntry.Location](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry-Location)
* `resolution`: See [ServiceEntry.Resolution](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry-Resolution)

This gets my app up and running. The k8s secret supplies the mongo connection string and the `ServiceEntry` permits egress from the istio mesh to my mongo cluster.


## Allowing external traffic to hit my app

By default, no traffic from outside the Istio service mesh is permitted in. In order to allow traffic to enter the cluster we must [configure ingress](https://istio.io/docs/tasks/traffic-management/ingress/#configuring-ingress-using-an-istio-gateway).  

A [Gateway](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#Gateway) sits at the edge of the service mesh and describes what traffic may enter (ports, protocols, etc). My gateway simply allows any http traffic on port 80:

{% highlight shell %}
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ctmd-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
{% endhighlight %}


The gateway just permits traffic to enter the cluster. A [VirtualService](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#VirtualService) is then needed to route that traffic to the correct service. It is bound to the gateway and forwards traffic arriving at the specified port / hosts. In my manifest, all http traffic arriving at the `ctmd-gateway` gateway with the url prefix `/api` is forwarded on to the `marketdata` k8s service on port `80`.

{% highlight shell %}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ctmd
spec:
  hosts:
  - "*"
  gateways:
  - ctmd-gateway
  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        port:
          number: 80
        host: marketdata  
{% endhighlight %}


### Testing

I can get the external ip address of my istio gateway from the GKE UI by looking for the "Load balancer" service "istio-ingressgateway". I can then point my browser to `http://<istio gateway>/api/v1/currencies` and see my list of currencies!

K8s and istio manifests are available on [github](https://github.com/MorganPeat/CryptoTracker/commit/45958c).


  
