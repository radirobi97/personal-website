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

![traffic-flow](/personal-website/images/traffic_flow.png)

In Kubernetes we host applications in a containerized manner. 
externalTrafficPolicy=local is an annotation on the Kubernetes service resource that can be set to preserve the client source IP. When this value is set, the actual IP address of a client (e.g., a browser or mobile application) is propagated to the Kubernetes service instead of the IP address of the node.