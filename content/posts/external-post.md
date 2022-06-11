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

The offical documentation says: ExternalTrafficPolicy denotes if this Service desires to route external traffic to node-local (only that node as the incoming traffic) or cluster-wide endpoints. Meaning, using this parameter we can decide whether we want to allow this "cross-node" routing. ExternalTrafficPolicy can be on of the followings:

* **Cluster**: The traffic hits one of the nodes, however the target POD is on a different node. Using Cluster mode as ExternalTrafficPolicy makes possible for kube-proxy to forward the traffic to a different node. See this on the figure below. 

![traffic-flow](/personal-website/images/traffic_flow.png)

* **Local**:  The traffic hits one of the nodes. In case of Local as ExternalTrafficPolicy kube proxy is only able to route the traffic locally. Meaning, only to PODs which are on the same node. 

![traffic-flow](/personal-website/images/local_external.png)

## Comparison of Local and Cluster

#### externalTrafficPolicy: Cluster
This is the default value for services. It provides an equally distributed traffic between PODs irregardles of where the PODs are located. However, this method could add extra network hops to the traffic if the target POD is on a different node. 
Also this mechanism introduces the usage of SNAT-ting (source network address translation). SNAT results in loss of IP of the client. 

You probably ask why SNAT is needed? Lets take what happens if we use only DNAT. Lets image a scenario where the Load Balacner forwards the traffic to a node however the target POD is on a different node. Thats why we use Cluster as externalTrafficPolicy. Here is the flow:

![traffic-flow](/personal-website/images/without_snat.png)

1. Client hits the Load Balancer and the LB will route the traffic to one of the nodes. 
2. On the node, kube-proxy handles the incoming traffic, modifes the destination in the packet and forwards the packet to the other node. On this node kube-proxy forwards the traffic to the destination POD. 




In Kubernetes we host applications in a containerized manner. 
externalTrafficPolicy=local is an annotation on the Kubernetes service resource that can be set to preserve the client source IP. When this value is set, the actual IP address of a client (e.g., a browser or mobile application) is propagated to the Kubernetes service instead of the IP address of the node.