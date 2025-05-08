---
layout: post
title:  "Kubernetes Explained: Understanding the Key Components Driving Modern Infrastructure ⚙️"
description: "An in-depth exploration of service mesh architecture — what it is, how it works, and why it's essential for modern microservices environments."
author: lorenzo-tettamati
categories: [ Cortexflow, kubernetes ]
image: "assets/images/data-center.webp"
tags: [featured]
---
Hi everyone, in today's episode we'll explore the Kubernetes key components. In the latest episode, we learned why Kubernetes is so important for software engineering and why almost every big company relies on it. Our journey will be supported with examples and illustrations to help everyone understand the essential functionalities of Kubernetes.

Let's dive into it! 🚢


## Glossary:
Here’s a list of common words and terms used in this article. This will help you form a basic understanding before reading the full article.

**Container:** 
A container is a standardized unit of software that packages an application’s code along with all its dependencies, libraries, and configuration files needed to run. Containers ensure that the software behaves consistently regardless of the environment. Common container technologies include Docker and Kubernetes.

**Distributed system:**
A distributed system is a collection of independent computers or nodes that work together to appear as a single coherent system to the end user. These systems communicate with one another over a network to coordinate tasks, share resources, and achieve common objectives.

**Daemon:**
In the field of container orchestration, a daemon refers to a background service or process that continuously runs and manages tasks without user intervention.

**Production Environment:**
The production environment is the setting where an application or service is made available to end-users. In this environment, the application is expected to operate stably, efficiently, and securely, often under real-world traffic conditions and with configurations optimized for performance.

**API:**
An API (Application Programming Interface) is a set of rules and definitions that allows different software applications to communicate with each other. APIs define how requests are sent, data is structured, and responses are received. APIs are commonly used for web services (e.g., REST, GraphQL), hardware, operating systems, and more.

**YAML:**
YAML (YAML Ain't Markup Language) is a human-readable data serialization format often used for configuration files. YAML uses hierarchical indentation to represent data structures like maps, lists, and scalar values. It is easy to read and write, making it popular in container configuration and resource orchestration.

## Architecture 
### What's a Kubernetes cluster?

A Kubernetes cluster is a set of machines, called nodes, that can run containerized applications. Kubernetes adopts a distributed architecture based on a client-server model to orchestrate the containers. The Kubernetes architecture is composed of two core components:

## Master Node

### The control plane
The brain of the cluster. It is responsible for managing the state of the cluster. In production environments,  the control plane usually runs on multiple nodes that span across several data center zones. The second is a set of worker nodes. These nodes run the containerized application workloads. The containerized applications run in a Pod. 

## _What's a Pod?_
Pods are the smallest deployable units in Kubernetes. A pod hosts one or more containers and provides shared storage and networking for those containers. Pods are created and managed by the Kubernetes control plane. They are the basic building blocks of Kubernetes applications. 

Now let’s dive a bit deeper into the control plane. It consists of several core components such as the API server, etc, scheduler, and controller manager. 

### API server
This is the primary interface between the control plane and the rest of the cluster. It exposes a RESTful API that allows clients to interact with the control plane and submit requests to manage the cluster. The API server is the gateway for the [kubectl](https://kubernetes.io/docs/reference/kubectl/) which is a command line tool for communicating with the the control plane using the API.

For example the command 

```
 kubectl run "pod-name" --image="image-name"
```
 is used to create a Pod with a desired _pod-name_ and with the desidered _image-name_

### etcd 
Etcd is an open-source distributed key-value store and plays an essential role in the Kubernetes control plane. It stores the cluster's persistent state. In Kubernetes etcd acts as the primary datastore, storing all the cluster data including all the configuration, state, and metadata. It is used by the API server to retrieve and update the cluster's state, ensuring that the actual state of the cluster matches the desired state defined by the users and the administrator.

_Fun fact: ETCD is composed of the words "etc" and "d." "etc" is derived from the UNIX directory "/etc," which houses configuration files, and "d" stands for "distributed"_

### Scheduler:
The scheduler is responsible for scheduling pods onto the worker nodes in the cluster. It uses information about the resources required by the pods and the available resources on the worker nodes to make placement decisions. 
In a cluster, Nodes that meet the scheduling requirements for a Pod are called feasible nodes. If none of the nodes are suitable, the pod remains unscheduled until the scheduler is able to place it.

Kube-scheduler selects a node for the pod in a 2-step operation:

- Filtering:
The filtering step identifies the nodes where the Pod can be scheduled. After this step, the node list contains the suitable nodes (usually more than one). If the list is empty, the Pod is not (yet) schedulable.


- Scoring:
In the scoring step, the scheduler ranks the remaining nodes to choose the best placement for the Pod. Each node that passed the filtering is given a score based on the active scoring rules.


Finally, kube-scheduler assigns the Pod to the Node with the highest ranking. If there is more than one node with equal scores, kube-scheduler selects one of these at random.


### Controller manager 
The controller manager is responsible for running controllers that manage the state of the cluster. The replication controller ensures that the desired number of replicas of a pod is running, the deployment controller manages the rolling update and rollback of deployments, and the endpoint controller manages the endpoint of the services.



![Image 1: A summary of the main components of the master node](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/chu8ty0y8ayhsqxgqfdv.png)

_Image 1: A summary of the main components of the master node_

## Worker Nodes
The core components of Kubernetes running on the worker nodes include the kubelet, container runtime, and kube-proxy.

### Kubelet
The kubelet is a daemon that runs on each worker node. It's the primary node-agent that runs on each node. It is responsible for communicating with the control plane. It can register the node with the apiserver using one of: the hostname; a flag to override the hostname; or specific logic for a cloud provider. It receives instructions from the control plane about which pods to run on the node, and ensures that the desired state of the pods is maintained.
The kubelet works in terms of a PodSpec. It takes a set of PodSpecs and ensures that the containers, described in those PodSpecs are running and healthy.
The Kubelet doesn't manage containers that were not created by Kubernetes.
Here's an example of a PodSpec:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: nginx-config
      configMap:
        name: nginx-config-map
  restartPolicy: Always

```
In this context the Kubelet reads the PodSpec and tells the container runtime to download the container image "nginx:latest", next it runs the containers and mounts the volume defined in spec.volumes. The kubelet allocates the resource defined in spec.resources and monitors the health of the container.

### Container Runtime 
The container runtime runs the containers on the worker nodes. It is responsible for pulling the container images from a registry. Start and stop the containers and manage the containers' resources. Kubernetes supports container runtimes such as containered, CRI-O, and any other implementation of the Kubernetes CRI (Container Runtime Interface). There are 2 types of Container Runtimes: Low-level Container Runtime and High-Level Container Runtimes. Since this is a broad topic I will expand on the concept of runtime containers in a future article.

### Proxy
The kube-proxy is a network proxy that runs on each worker node. It is responsible for routing traffic to the correct pods. It also provides load balancing for the pods and ensures that traffic is distributed evenly across them. After kube-proxy is installed it authenticates with the API server and when new services or endpoints are added or removed the API server communicates these changes to the kube-proxy, then kube-proxy applies these changes as NAT rules. When traffic is sent to a service it's redirected to a backend Pod based on these rules. As for the Container Runtimes this is a small introduction about what's the kube-proxy and what are his main functionalities and I'll expand on all these concepts in a dedicated article.


![Image2: A summary of the main components of a kubernetes cluster that includes the master node and two worker nodes. In the first worker node there are 5 pods running microservices and the second worker node there are 3 pods running microservices](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/px98hha739igybxr65pl.png)

_Image2: A summary of the main components of a kubernetes cluster that includes the master node and two worker nodes. In the first worker node there are 5 pods running microservices and the second worker node there are 3 pods running microservices_

## Conclusion

Kubernetes components are a beautiful example of technology and sophisticated infrastructure. Today we explored how the core components work and interact to enable efficient application deployment and scaling.
Thank you for reading! 🙏

