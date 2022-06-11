+++
author = "Robert Radi"
date = "2020-12-03"
title = "Effect of externalTrafficPolicy on Kubernetes services"
tags = [
    "kubernetes",
    "service",
]
categories = [
    "kubernetes",
    "in-depth",
]
+++

## Just to be on the same page
Kubernetes is all about containerization. Every container is placed inside a POD. The scheduler places the PODs onto different nodes. Nodes are the physical/virutal machines. 
A POD can have one ore more replicas by using ReplicaSet. There are no guarantees for that the PODs of the same ReplicaSet will be hosted on the same node. 

## How does the external traffic reach the destination POD?
First of all the traffic hits one of the nodes. This node will route the traffic to the target POD via kube-proxy. However, this POD could be on the node where the traffic came in, or could be on a different node. This is where ExternalTrafficPolicy comes into picture. 

The offical documentation says: ExternalTrafficPolicy denotes if this Service desires to route external traffic to node-local or cluster-wide endpoints. Meaning, using this parameter we can decide whether we want to allow this "cross-node" routing. ExternalTrafficPolicy can be on of the followings:

* **Cluster**: The traffic hits one of the nodes, however the target POD is on a different node. Using Cluster mode as ExternalTrafficPolicy makes possible for kube-proxy to forward the traffic to a different node. See this on the figure below. 

![traffic-flow](/personal-website/images/traffic_flow.png)
(/personal-website/images/disk_export.webp#center)

* 



In Kubernetes we host applications in a containerized manner. 
externalTrafficPolicy=local is an annotation on the Kubernetes service resource that can be set to preserve the client source IP. When this value is set, the actual IP address of a client (e.g., a browser or mobile application) is propagated to the Kubernetes service instead of the IP address of the node.