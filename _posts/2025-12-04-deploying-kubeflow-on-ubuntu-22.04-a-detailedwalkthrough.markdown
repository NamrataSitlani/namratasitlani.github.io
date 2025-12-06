---
title: "Deploying Kubeflow on Ubuntu 22.04: A Detailed Walkthrough"
layout: post
date: 2025-12-04
image: /assets/images/lxd_kubernetes.jpg
headerImage: false
tag:
- Kubeflow
- Kubernetes
- juju
- microk8s
star: true
category: blog
author: Namrata Sitlani
description: Markdown summary with different options
---


## What is Kubeflow
Kubeflow is a tool that helps you build, run, and manage machine-learning workflows on Kubernetes.

Think of it like this:

* Kubernetes manages containers.

* Kubeflow manages ML tasks on top of Kubernetes.

So instead of manually training models, deploying notebooks, running pipelines, and managing data manually,
Kubeflow makes everything organized, automated, and easier to scale.


## Kubeflow Installation Procedure: Step-by-Step
We’ll be deploying Kubeflow using Juju and MicroK8s together.
MicroK8s will act as our lightweight Kubernetes cluster, and Juju will handle the deployment and management of all the Kubeflow components.
Instead of manually creating each service, Juju automates the setup, connects everything for us,
and makes the whole process much easier and more reliable.

1. Install MicroK8s

   MicroK8s provides a minimal, easy-to-run Kubernetes environment suitable for local labs and development setups.
   To install it, run:

   ```bash
   sudo snap install microk8s --classic
   ```

2. Install Juju

   Juju will handle the actual deployment of Kubeflow for us. You can install it with: 

   ```bash
   sudo snap install juju --classic
   ```

3. Verify MicroK8s Installation

   The MicroK8s setup may take a few minutes to finish. You can check when it’s ready by running:

   ```bash
   microk8s status --wait-ready
   ```

   If you run into permission errors while checking MicroK8s status, fix by running following
   commands:

   ```bash
   sudo usermod -a -G microk8s training
   sudo chown -R training ~/.kube

   After this, reload the user groups either via a reboot or by running 'newgrp microk8s'.
   ```
   Recheck MicroK8s status:

   ```bash
   microk8s status --wait-ready
   ```
   ![MicroK8s status](/assets/images/microk8s_status.png)

4. Set Up a kubectl Alias

   To make working with MicroK8s easier, create a shortcut so you can use kubectl directly:

   ```bash
   alias kubectl="microk8s kubectl"
   ```
   This lets you run all your Kubernetes commands without typing microk8s each time.

   Verify Kubernetes pods:

   ![Kubernetes Pods](/assets/images/kubernetes_status.png)


5. Enable DNS and Storage for Kubernetes

   Kubernetes needs DNS and basic storage to function smoothly. Enable both with a single command:

   ```bash
   microk8s enable dns storage
   ```
   ![Micork8s Add-ons](/assets/images/microk8s_addons.png)

   Verify pods have been created:

   ![Kubernetes Pods Storage](/assets/images/kubernetes_status_addon.png)

6. Verify Juju Models and Controllers

   Before deploying anything, it’s a good idea to check that there are no existing Juju models or controllers running:

   ```bash
   juju models
   juju controllers
   ```

7. Connect Juju to Your Kubernetes Cluster

   First, export the MicroK8s configuration so kubectl and Juju can communicate with your cluster:

   ```bash
   microk8s config > ~/.kube/config
   ```
   Next, register your Kubernetes cluster with Juju as a new cloud:

   ```bash
   juju add-k8s myk8s
   ```
   ![Connect Juju](/assets/images/configure_juju.png)

8. Set Up a Juju Controller

   Next, bootstrap a Juju controller to manage your deployments on the Kubernetes cluster:

   ```bash
   juju bootstrap myk8s my-controller
   ```
   ![Juju Controller](/assets/images/juju_bootstrap.png)

   Verify the controller:

   ```bash
   juju controllers 
   kubectl get po -A
   ```
   ![Juju List Controller](/assets/images/juju_controllers.png)
   ![Controller Pods](/assets/images/controller_pods.png)

9. Add Kubeflow Model

   Create the model for kubeflow:

   ```bash
   juju add-model kubeflow
   ```
   You will see an output like the following:

   ```
   Added 'kubeflow' model on myk8s/localhost with credential 'myk8s' for user 'admin'
   ```

   Verify the model:

   ![Kubeflow Model](/assets/images/my_controller.png)
  

10. Deploy Kubeflow

    Deploy Kubeflow using juju:

    ```bash
    juju deploy kubeflow
    ```

    If Juju reports a trust-related error during deployment, rerun the command with explicit trust enabled:

    ```bash
    juju deploy kubeflow --trust
    ```

11. Set Up the Kubeflow Dashboard Login

    Kubeflow uses an email address and password for authentication.
    You can configure these securely using Juju without exposing the password in your terminal history.

    Set the dashboard email:

    ```bash
    juju config dex-auth static-username="admin@example.com"
    ```
    Enter password securely:

    ```bash
    read -s -p "Enter Kubeflow password: " pwd
    echo
    juju config dex-auth static-password="$pwd"
    ```

12. Check the Istio Gateway Status

    Verify whether the Istio gateway is stuck in a Pending state by inspecting the Kubeflow services:

    ```bash
    kubectl get services -n kubeflow | grep istio
    ```

13. Install MetalLB to enable the gateway:

    ```bash
    microk8s enable metallb:192.168.122.202-192.168.122.220
    ```
   ![Enable Metallb](/assets/images/metallb.png)

14. Access Kubeflow

    After enabling MetalLB, check again to confirm that the previously Pending services now have an assigned external IP:

    ``` bash
    kubectl get services -n kubeflow | grep istio
    ```
    Access the Kubeflow dashboard locally using the following URL:

    ```bash
    http://192.168.122.202
    ```

    You should now see the Kubeflow login page.
    Log in using the static Dex credentials you configured earlier.

  ![Kubeflow Login](/assets/images/login.png)
  ![Kubeflow Dashboard](/assets/images/dashboard.png)

