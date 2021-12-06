---
layout: post
title:  "Digital Ocean Kubernetes Challenge : Setting up crossplane on a DO Managed kubernetes instance"
date: 2021-12-06 17:00:00 +0100
categories: Tech
---

This blog post is part of the [Digital Ocean Kubernetes Challenge](https://www.digitalocean.com/community/pages/kubernetes-challenge) and about how to setup the still in development [DO-Crossplane Provider](https://github.com/crossplane-contrib/provider-digitalocean). This guide should provide useful to people as the current documentation is still lacking.
Big shoutout to [@kimschles](https://twitter.com/kimschles?lang=en) for helping me figure part of this out.
#### Setup DO K8s Cluster and have kubectl setup to connect to the cluster
First we need to setup a digital ocean Kubernetes cluster, and use the doctl tooling to automatically generate the relevant kubectl configuration, the digital ocean GUI guides you through how to do this.
#### Install Basic Crossplane
{% highlight Bash %}
kubectl create namespace crossplane-system
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system crossplane-stable/crossplane

#Check that it installed correctly
helm list -n crossplane-system

kubectl get all -n crossplane-system
{% endhighlight %}

#### Next install crossplane CLI
Note this is how the offical crossplane CLI documentation recomends installing it, you should never pipe directly into sh, see [this](https://www.seancassidy.me/dont-pipe-to-your-shell.html) as to why you shouldn't pipe into shell.
{% highlight Bash %}
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/release-1.5/install.sh | sh 
{% endhighlight %}

#### Install the Custom Resource Definitions for the DO Provider
{% highlight Bash %}
git clone https://github.com/crossplane-contrib/provider-digitalocean.git
kubectl apply -f provider-digitalocean/package/crds -R
{% endhighlight %}

#### Next install the latest version of the digital ocean provider
{% highlight Bash %}
kubectl crossplane install provider crossplane/provider-digitalocean:latest
{% endhighlight %}

#### In a new terminal window launch the provider
{% highlight Bash %}
go run cmd/provider/main.go --debug 
{% endhighlight %}

#### Deployment
Now we are finally ready to start deploying
First we need to create an ProviderConfig CRD and a Provider secret in the `crossplane-system` namespace.
This is done by creating the following `provider.yml` file where the `data:token:` is a base64 encoded version of a digital ocean api token with read/write permissions
{% highlight yaml %}
apiVersion: v1
kind: Secret
metadata:
  namespace: crossplane-system
  name: provider-do-secret
type: Opaque
data:
  token: VXNlIHlvdXIgb3duIGRpZ2l0YWwgb2NlYW4gc2VjcmV0IHRva2VuIHNpbGx5IDpQ
---
apiVersion: do.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: example 
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: provider-do-secret
      key: token
{% endhighlight %}
Then once the file `provider.yml` is created, it can be applied to the cluster via
`kubectl -n crossplane-system apply -f provider.yml`

Finally we are ready to deploy digital ocean infrastructure! Currently the do-provider only supports droplets and load balancers.

Droplets can then be spun up like the `provider.yml` file by creating a yml file with the following schema.

{% highlight yaml %}
compute.do.crossplane.io/v1alpha1
kind: Droplet
metadata:
  name: example
  annotations:
    crossplane.io/external-name: crossplane-droplet
spec:
  forProvider:
    region: nyc1
    size: s-1vcpu-1gb
    image: ubuntu-20-04-x64
  providerConfigRef:
    name: example
#The name: example above refers to the name of the provider config CRD
{% endhighlight %}

and again applying it with `kubectl -n crossplane-system apply -f droplet.yml`

Loadbalancers follow the following schema
{% highlight yaml %}
apiVersion: loadbalancer.do.crossplane.io/v1alpha1
kind: LB
metadata:
  name: example-lb
spec:
  forProvider:
    region: nyc1
    algorithm: round_robin
    healthCheck:
      interval: 300
      timeout: 300
      unhealthyThreshold: 10
      healthyThreshold: 10
  providerConfigRef:
    name: example
{% endhighlight %}

There you have it! Thats how you install and use the crossplane digital ocean provider to manage DO droplets and loadbalancers.
Now obviously this software is still in the very early stages so I personally wouldn't reccomend using it for production, but exciting to see how it evolves!
