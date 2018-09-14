---
layout: post
title:  "Microservice spike part 5 - SSL termination"
date:   2018-09-12 12:00:00
categories: [Microservices]
tags: [cloud, k8s]
draft: false
---

This is part 5 in a series of posts where I spike out a cloud microservice app on GCP. In this post I secure the endpoint for my app by configuring it to use SSL / TLS (HTTPS).

* [Part 1 - the CryptoTracker app]({% post_url 2018-08-21-cryptotracker-a %}) introduces my prototype microservice app
* [Part 2 - run in GCP]({% post_url 2018-08-21-cryptotracker-b %}) gets the app running in Docker in Google Compute Engine
* [Part 3 - run in GKE]({% post_url 2018-08-23-cryptotracker-c %}) gets the app running in Google Kubernetes Engine
* [Part 4 - run in an Istio service mesh]({% post_url 2018-08-30-cryptotracker-d %}) gets the app running in an Istio service mesh
* [Part 5 - secure the app via TLS]({% post_url 2018-09-12-cryptotracker-e %}) (this page) secures the app by using TLS over HTTPS

## Securing ingress

There is a great document on the isito site describing how to [secure ingress with HTTPS](https://istio.io/docs/tasks/traffic-management/secure-ingress/) which I followed in order to secure my app. It was simple enough to follow, but below are a few details that helped me.  In a nutshell, the steps are:
1. Configure istio listen for HTTPS traffic using a X.509 certificate and private key  
1. Forward that traffic on to my app via HTTP


### Generate certificates and keys
The istio docs use [a script](https://github.com/nicholasjackson/mtls-go-example/blob/master/generate.sh) to generate all the necessary public / private key pairs. The domain name I used to generate the certs was `cryptotracker-demo.com`. This name is stamped into my certificates as the [Common Name](https://www.ssl.com/faqs/common-name/) (CN). A client will access my app via a host name (eg. `http://cryptotracker-demo.com/`) and will only trust my site if the server certificate's CN matches the expected host name.  

The istio server uses the _application_ certificate which was signed by the _intermediate_. The client will have the _intermediate_ public key and trust any certificates signed by it, so it should trust my app.



### Create TLS secret

{% highlight shell %}
kubectl create -n istio-system secret tls istio-ingressgateway-certs --key cryptotracker-demo.com/3_application/private/cryptotracker-demo.com.key.pem --cert cryptotracker-demo.com/3_application/certs/cryptotracker-demo.com.cert.pem
{% endhighlight %}

A [kubernetes TLS secret](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-secret-tls-em-) will hold the public / private key pair for the application certificate. In order that the istio ingress gateway can read the secret it must be created in the correct `istio-system` namespace. The k8s deployment manifest for the istio ingress gateway shows it maps the volume `ingressgateway-certs` from a secret named `istio-ingressgateway-certs` and mounts the certificates to `/etc/istio/ingressgateway-certs`. They will be mounted as `tls.crt` and `tls.key`.


### Create istio TLS gateway

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
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
      privateKey: /etc/istio/ingressgateway-certs/tls.key
    hosts:
    - "cryptotracker-demo.com"
{% endhighlight %}

This [Gateway](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#Gateway) manifest instructs the isito ingress load balancer to listen on port 443. The `tls` section instructs the gateway to use the public / private key pair that I injected earlier via a tls secret. The `hosts` section is used during the TLS handshaking process: the client supplies the host name as a [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) value and istio will use it to match the client to the correct istio gateway.  


As I did [before]({% post_url 2018-08-30-cryptotracker-d %}), a [VirtualService](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#VirtualService) bridges traffic from the gateway to my app on HTTP port 80.

{% highlight shell %}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ctmd
spec:
  hosts:
  - "cryptotracker-demo.com"
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



### Test it

Following the istio docs I used `curl` to test.
{% highlight shell %}
curl -v -HHost:cryptotracker-demo.com --resolve cryptotracker-demo.com:443:http://35.195.72.44 --cacert cryptotracker-demo.com/2_intermediate/certs/ca-chain.cert.pem https://cryptotracker-demo.com:443/api/v1/currencies
{% endhighlight %}

The `Host` header and `--resolve` parameters are used to supply the correct [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) value which will cause istio to match my request to my app. They are also used by `curl` to verify that the server's host name is correct, by matching against the certificate CN.  

The _intermediate_ certificate is used to verify the server's identity. As described above, this is fine since the server's certificate was signed by the intermediate.

</p>

It all works, but as a further test I used a new, blank, VM in a different GCP zone. Once I had created a basic VM I copied the intermediate certificate onto it by using [SCP](https://cloud.google.com/compute/docs/instances/transfer-files#linux) from the cloud shell:

{% highlight shell %}
gcloud compute scp cryptotracker-demo.com/2_intermediate/certs/ca-chain.cert.pem instance-1:~/
{% endhighlight %}

I could then use the same `curl` command to verify I could connect. In the real world one wouldn't use self-signed certificates of course, my app would use a certificate signed by a trusted CA that is already known to my machine. I'd also have a DNS name so the `Host` header and SNI value would be passed automatically.
