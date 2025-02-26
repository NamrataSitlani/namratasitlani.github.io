---
title: "How to Deploy MySQL on Kubernetes"
layout: post
date: 2025-02-26
image: /assets/images/mysql.jpg
headerImage: false
tag:
- MySQL
- Kubernetes
star: true
category: blog
author: Namrata Sitlani
description: Markdown summary with different options
---


![<img>](/assets/images/mysql.jpg)


**How to Deploy MySQL on Kubernetes: A Step-by-Step Guide**

Kubernetes is an excellent tool for orchestrating containers, and one common use case is deploying databases like MySQL. If you're setting up MySQL in a Kubernetes environment, here's a detailed guide on how to do it, from installing `kubectl` to deploying MySQL with persistent storage.

### Step 1: Install `kubectl`

First, you'll need to install the Kubernetes command-line tool, `kubectl`, which will allow you to interact with your Kubernetes cluster.

Run the following command to install `kubectl` on your system:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Step 2: Create a Kubernetes Secret for MySQL Password

For security reasons, you shouldn’t store sensitive information like database passwords in plain text. Kubernetes Secrets are perfect for this.

To create a MySQL password, run the following:

```bash
echo -n `pwgen 32 1` > ~/password
kubectl create secret generic mysql-secret --from-file=password
```

This creates a secret called `mysql-secret` containing the password for MySQL.

### Step 3: Set Up Persistent Storage

MySQL requires persistent storage, so we'll create a Persistent Volume (PV) and Persistent Volume Claim (PVC) to store MySQL data.

Create a file named `mysql-storage.yaml` with the following contents:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

Apply the configuration:

```bash
kubectl apply -f mysql-storage.yaml
```

This will create the Persistent Volume and the associated Persistent Volume Claim.

### Step 4: Deploy MySQL on Kubernetes

Now it's time to deploy MySQL. We'll create a Deployment to manage the MySQL pod and a Service to expose it.

Create a file named `mysql-deployment.yaml` with the following contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:latest
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
```

Deploy the MySQL service and pod:

```bash
kubectl apply -f mysql-deployment.yaml
```

### Step 5: Verify Deployment

To make sure everything is running, check the status of your pods:

```bash
kubectl get pods
```

You should see something like this:

```
NAME                     READY   STATUS    RESTARTS   AGE
mysql-6d7f5d5944-pmg2g   1/1     Running   0          17s
```

### Step 6: Access the MySQL Pod

To interact with MySQL, you'll need to access the MySQL pod. Use `kubectl exec` to open a bash shell inside the pod:

```bash
kubectl exec -it mysql-6d7f5d5944-pmg2g -- /bin/bash
```

Once inside, use the following command to access the MySQL shell:

```bash
mysql -p
```

When prompted, enter the password you created earlier in the Kubernetes secret. You should see the MySQL shell:

```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 9.2.0 MySQL Community Server - GPL
...
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)
```

### Conclusion

That’s it! You’ve successfully deployed MySQL on Kubernetes with persistent storage. This setup ensures your MySQL data remains intact even if the pod is restarted. With Kubernetes handling the orchestration, it’s much easier to manage and scale MySQL deployments as your application grows.

