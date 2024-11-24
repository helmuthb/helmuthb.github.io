---
layout: post
title: "Adding an ingress controller to a k0s cluster"
date: 2024-11-24
categories: kubernetes cloud
---

# Adding an ingress controller to a k0s cluster

Without an ingress controller, we can't expose our services to the internet.

Therefore we will setup the [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) for our cluster.

_This is a continuation of the previous post, [Setting up a k0s cluster on netcup](/posts/2024-11-23-k0s-netcup-part1)._

## Why nginx?

NGINX is a popular and well-known ingress controller, and is often used for production environments.

There are alternatives, like [Ambassador](https://www.getambassador.io/) and [Traefik](https://traefik.io/), and they do offer a more complete package like API management or integrated certificate provisioning.
Still, NGINX is a popular choice, and its setup is not complicated either.

## Prerequisites

For this setup, we'll need:
- Our k0s cluster (see [Setting up a k0s cluster on netcup](/posts/2024-11-23-k0s-netcup-part1))

## Installation Steps

### 1. Prepare local machine

We will need a local installation of `helm`.
On my MacBook I used `homebrew`:

```bash
brew install helm
```

### 2. Install the ingress controller

We will use the NGINX Helm Chart for installation. First we add the repo:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

In the usual setup, the Ingress Controller will request public IPs whenever needed from the cloud provider.

In our case, we only have the static IPs of the VMs available.
Therefore, we instruct nginx to use the host network instead of the public network, and to use a DaemonSet instead of a Deployment.

This is the command I used:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --create-namespace \
    --namespace ingress-nginx \
    --set controller.hostNetwork=true \
    --set controller.service.type="" \
    --set controller.type=DaemonSet
```

### 3. Install the cert-manager

We will use the Jetstack Helm Chart for installation. First we add the repo:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

For installing the cert-manager I used this command:

```bash
helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --set crds.enabled=true
```

Next, I added the TLS resolvers for both staging and production Let's Encrypt service.

`letsencrypt-dev.yml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dev
spec:
  acme:
    email: me@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-dev-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

`letsencrypt-prod.yml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: me@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

And then I created the resources on Kubernetes:

```bash
kubectl create -f letsencrypt-dev.yml
kubectl create -f letsencrypt-prod.yml
```

## Next Steps

We have a Kubernetes cluster with an ingress controller and a cert-manager setup.

In the next post we will add [Longhorn](https://longhorn.io/) to our cluster for persistent storage.

_Thanks for reading!_