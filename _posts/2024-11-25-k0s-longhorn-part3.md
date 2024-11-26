---
layout: post
title: "k0s on NetCup Part 3: Adding Longhorn Storage"
date: 2024-11-25
categories: kubernetes storage
---

# k0s on NetCup Part 3: Adding Longhorn Storage

After setting up our k0s cluster and configuring ingress in the previous posts, let's add persistent storage capabilities using [Longhorn](https://longhorn.io/). Longhorn is a lightweight and reliable distributed block storage system, perfect for our k0s cluster.

_This is a continuation of the previous posts, [Setting up a k0s cluster on netcup](/posts/2024-11-23-k0s-netcup-part1) and [ Adding an ingress controller to a k0s cluster](/posts/2024-11-24-k0s-ingress-part2)._

## Why Longhorn?

Longhorn is an excellent choice for our k0s cluster because:

- Distributed storage with data replication
- Backup and restore capabilities
- Simple installation and management
- Low resource overhead
- Perfect for small to medium clusters

## Prerequisites

The nodes we have from netcup have sufficient disk space (for some small experiments at least).
However, we need to make sure that each node has the package `open-iscsi` installed, which is needed for Longhorn to work.

```bash
# On each node:
apt-get update
apt-get install -y open-iscsi
```

We will also need a DNS entry for the Longhorn UI, for example using the name `longhorn.helmuth.at`.
I create this entry in the DNS provider as a CNAME pointing to `k0s-cluster.helmuth.at`.

## Installing Longhorn

We'll use the helm chart to install Longhorn:

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system
```

This will create the `longhorn-system` namespace and deploy all necessary components. Monitor the deployment:

```bash
kubectl -n longhorn-system get pods
```

Wait until all pods are running.

## Accessing the Longhorn UI

To access the Longhorn UI securely, we'll create an ingress resource. First, let's create a Basic Auth secret:

```bash
# Generate auth file
USER=admin               # replace with your desired username
PASSWORD=secret-password # replace with your desired password
echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" > auth

# Create secret
kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
```

_It's important to have the secret key named "auth" - this is the name of the file we created._

Now create the ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
    nginx.ingress.kubernetes.io/proxy-body-size: 10000m
spec:
  rules:
  - host: longhorn.helmuth.at
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
  tls:
  - hosts:
    - longhorn.helmuth.at
    secretName: longhorn-tls
```

Save this as `longhorn-ingress.yaml` and apply:

```bash
kubectl apply -f longhorn-ingress.yaml
```

## Testing Longhorn

Let's create a test deployment using Longhorn storage. For this, I will deploy MySQL with persistent storage.
First the PVC (Persistent Volume Claim):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

Then the deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8.0.23
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: test@123
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

And finally the service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  selector:
    app: mysql
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
```

Apply these configurations:

```bash
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
```

Now we can access the MySQL database from the cluster.

```bash
kubectl run mysql-client --image=mysql:5.7 -it --rm --restart=Never -- mysql -h mysql -u root -p
# No prompt visible - enter password: test@123
mysql> show databases;
mysql> quit
```

## Next Steps

We have a Kubernetes cluster with an ingress controller, a cert-manager setup, and persistent storage using Longhorn.
We are ready for some experiments!

_Thanks for reading!_