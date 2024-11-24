---
layout: post
title: "Setting up a k0s cluster on netcup"
date: 2024-11-23
categories: kubernetes cloud
---

# Setting up a k0s cluster on netcup

For experiments and learning purposes I always look for various ways to run Kubernetes workloads.
The easiest way is using a cloud provider and their Kubernetes service, but experimenting with various Kubernetes distributions is valuable as well.

In this post, I'll show you how to set up a lightweight Kubernetes cluster using k0s on [netcup](https://www.netcup.com/en) VMs, a German hosting provider with a focus on affordability.

## Why k0s?

[k0s](https://k0sproject.io/) is a lightweight Kubernetes distribution that's perfect for both learning and production environments. Some key benefits include:

- Single binary installation
- Minimal resource requirements
- Built-in container runtime (containerd)
- Simple configuration
- Production-ready security settings

## Why netcup?

When looking for VMs for side-projects or experiments I discovered the offers from netcup.

A little bit hidden is an offer of a 2GB VPS for just 2 EUR per month: [VPS nano G11s 6M](https://www.netcup.com/de/server/vps/vps-nano-g11s-6m)
One customer can only order five, but that's still sufficient for experiments with creating a Kubernetes cluster.

Our cluster will use three VMs, which makes a monthly expense of just 6 EUR.

## Prerequisites

For this setup, we'll need:
- 3 netcup VPS (1 control plane + worker, 2 worker nodes)
- Debian 12 (Bookworm) on all nodes
- SSH access to all nodes
- Root privileges

## Installation Steps

### 1. Prepare the Nodes

When ordering the VPS from netcup, they come with Debian 12 preinstalled, including all current updates.

I have created DNS entries for the machines, using my domain.
You can also use the names provided by netcup if you prefer that.

In my case I named the machines like this:

* VPS #1 - used for control plane and as a worker: `k0s.helmuth.at`
* VPS #2 - used as a worker: `k0s-node1.helmuth.at`
* VPS #3 - used as a worker: `k0s-node2.helmuth.at`

I also ensured that the machines have the correct hostname, by editing the file `/etc/hostname` and adding the corresponding hostname to it.

We will use `k0sctl` for installing the nodes.
You'll have to install your SSH key onto the machines.
From your local machine:

```bash
ssh-copy-id root@k0s.helmuth.at
ssh-copy-id root@k0s-node1.helmuth.at
ssh-copy-id root@k0s-node2.helmuth.at
```

### 2. Prepare your local machine

You will have to install `k0sctl` on your local machine.
See https://github.com/k0sproject/k0sctl#installation for details.
On my MacBook I used `homebrew`:

```bash
brew install k0sproject/tap/k0sctl
```

In addition, you need to have `kubectl` available on your local machine as well.
If you have it installed already you can continue to use it, as long as it is current enough (see the [version skew policy](https://kubernetes.io/releases/version-skew-policy/) for details).

### 2. Create the `k0sctl` config file

We start with creating an initial `k0sctl` config file.
Run `k0sctl init` and save the file to your local machine:

```bash
k0sctl init > k0sctl.yaml
```

Next we adjust the config file to use the netcup VMs, and to use the first node as a worker as well.
You can use hostnames for the nodes, no need to use the IP addresses.

This was the config file I used:

```yaml
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s-cluster
spec:
  hosts:
  - ssh:
      address: k0s.helmuth.at
      user: root
      port: 22
      keyPath: null
    role: controller+worker
    noTaints: true
  - ssh:
      address: k0s-node1.helmuth.at
      user: root
      port: 22
      keyPath: null
    role: worker
  - ssh:
      address: k0s-node2.helmuth.at
      user: root
      port: 22
      keyPath: null
    role: worker
```

### 3. Create the Cluster

Thanks to `k0sctl`, creating the cluster is simple:

```bash
k0sctl apply
```

The cluster will be created with a single control plane + worker, and two additional worker nodes.

### 4. Load the config

We want to use the Kubernetes cluster from our local machine, therefore we load the config into our `.kube/config` file.

**Warning!**: the following command will replace your config - no not use it if you have already a Kubernetes cluster configured there!

```bash
k0sctl kubeconfig > ~/.kube/config
```

### 5. Check the cluster

On your local machine, check the nodes:

```bash
kubectl get nodes
```

## Next Steps

Your cluster is up and running!

In the next post we will setup an `nginx` ingress, together with `cert-manager`, for automatic creation of free, trusted TLS certificates from _Let's Encrypt_.

_Thanks for reading!_