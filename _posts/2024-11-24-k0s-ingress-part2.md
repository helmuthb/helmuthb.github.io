---
layout: post
title: "Adding an Ingress Controller to a k0s Cluster"
date: 2024-11-24
categories: kubernetes cloud
---

# Adding an Ingress Controller to a k0s Cluster

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

### 4. Prepare DNS records

I'm assuming you have a public domain under your control, like `helmuth.at`.

For the ingress I created a set of A records for the name `k0s-cluster.helmuth.at`, pointing to the three IP addresses of the nodes.

```
k0s-cluster A 1.2.3.1
k0s-cluster A 1.2.3.2
k0s-cluster A 1.2.3.3
```

Now for each hostname on the cluser I will just use a CNAME to `k0s-cluster.helmuth.at`, and that will give me a DNS-based load balancing of the three nodes.

_A DNS-based load balancing is a suboptimal option, as it depends on clients' behavior._
_But since we don't have a real load balancer in front of the nodes it's the best we can achieve._

### 5. Test the ingress and cert-manager

For a quick test we will create a CNAME `apache.helmuth.at` pointing to `k0s-cluster.helmuth.at`:

```
apache CNAME k0s-cluster.helmuth.at
```

Next we create a simple `apache` pod (no need for a deployment here as we just want to test the ingress controller) with a service attached to it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apache
  labels:
    app: apache
spec:
  containers:
  - name: apache
    image: httpd:2.4
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: apache
spec:
  selector:
    app: apache
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

Save this as `apache.yaml` and apply:

```bash
kubectl apply -f apache.yaml
```

With the service created, we can now add an ingress rule to the ingress controller:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apache-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-dev
spec:
  ingressClassName: nginx
  rules:
  - host: apache.helmuth.at
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: apache
            port:
              number: 80
  tls:
  - hosts:
    - apache.helmuth.at
    secretName: apache-tls
```

Wait maybe a minute, and then using an incognito tab in your browser, try to access `apache.helmuth.at` and check that the certificate is from the Let's Encrypt staging server.

_This means the browser won't trust it, but that's fine for now._

You should see on the certificate the issuer _Common Name_ `(STAGING) Wannabe Watercress R11` and the issuer _Organization_ `(STAGING) Let's Encrypt`.

If that is working we can update the ingress rule to use the production issuer:

```yaml
...
    cert-manager.io/cluster-issuer: letsencrypt-prod
...
```

Again, after a minute, when we now try to access the ingress in a new incognito tab, we should see no warnings from the browser, and a "It works!" message from our Apache web server.

_Why incognito tabs? The browser might cache the certificate. As we were using a staging certificate first, we did not want it to cache it._

## Next Steps

We have a Kubernetes cluster with an ingress controller and a cert-manager setup, and it works!

In the next post we will add [Longhorn](https://longhorn.io/) to our cluster for persistent storage.

_Thanks for reading!_