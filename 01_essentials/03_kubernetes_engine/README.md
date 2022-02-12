# Kubernetes Engine

Google Kubernetes Engine (GKE) provides a managed environment for deploying, managing, and scaling your containerized applications using Google infrastructure. The Kubernetes Engine environment consists of multiple machines (specifically Compute Engine instances) grouped to form a container cluster. 

Google Kubernetes Engine (GKE) clusters are powered by the Kubernetes open source cluster management system. Kubernetes provides the mechanisms through which you interact with your container cluster. You use Kubernetes commands and resources to deploy and manage your applications, perform administrative tasks, set policies, and monitor the health of your deployed workloads.

## Set a default compute zone

Active account name
```bash
gcloud auth list
```

List the project ID

```bash
gcloud config list project
```

Set default compute zone

```bash
gcloud config set compute/zone us-central1-a
```

## Create a GKE cluster

A cluster consists of at least one cluster master machine and multiple worker machines called nodes. Nodes are Compute Engine virtual machine (VM) instances that run the Kubernetes processes necessary to make them part of the cluster.

> Note: Cluster names must start with a letter and end with an alphanumeric, and cannot be longer than 40 characters.


Create a cluster with the following command:

```bash
gcloud container clusters create my-cluster
```

## Get authentication credentials for the cluster

Authentication credentials are needed in order to interact with the cluster.
To authenticate the cluster, run the following command

```bash
gcloud container clusters get-credentials my-cluster
```

## Deploy an application to the cluster

Let's deploy `hello-app` in the cluster.

GKE uses Kubernetes objects to create and manage your cluster's resources. Kubernetes provides the Deployment object for deploying stateless applications like web servers. Service objects define rules and load balancing for accessing your application from the internet.

To create a new Deployment `hello-server` from the `hello-app` container image, run the following `kubectl` create command:

```bash
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
```

This Kubernetes command creates a Deployment object that represents `hello-server`. In this case, `--image` specifies a container image to deploy. The command pulls the example image from a Container Registry bucket. `gcr.io/google-samples/hello-app:1.0` indicates the specific image version to pull. If a version is not specified, the latest version is used.

To create a Kubernetes Service, which is a Kubernetes resource that lets you expose your application to external traffic, run the following `kubectl expose` command:

```bash
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
```
In this command:

* `--port` specifies the port that the container exposes.
* `--type=LoadBalancer` creates a Compute Engine load balancer for your container.

To inspect the hello-server Service, run `kubectl get`:

```bash
kubectl get service
```
>Note: It might take a minute for an external IP address to be generated. Run the previous command again if the EXTERNAL-IP column status is pending.

To view the application from your web browser, open a new tab and enter the following address, replacing [EXTERNAL IP] with the EXTERNAL-IP for hello-server.

`http://[EXTERNAL-IP]:8080`

## Deleting the cluster
To delete the cluster, run the following command:

```bash
gcloud container clusters delete my-cluster
```