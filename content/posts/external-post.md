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

![traffic-flow](/personal-website/images/traffic_flow.png#center)

* **Local**:  The traffic hits one of the nodes. In case of Local as ExternalTrafficPolicy kube proxy is only able to route the traffic locally. Meaning, only to PODs which are on the same node. 

![traffic-flow](/personal-website/images/local_external.png#center)

## Comparison of Local and Cluster

#### externalTrafficPolicy: Cluster
This is the default value for services. It provides an equally distributed traffic between PODs regardless of where the PODs are located. However, this method could add extra network hops to the traffic if the target POD is on a different node. 
Also this mechanism introduces the usage of SNAT-ting (source network address translation). SNAT results in loss of IP of the client. 

You probably ask why SNAT is needed? Lets take what happens if we use only DNAT. Lets image a scenario where the Load Balacner forwards the traffic to a node however the target POD is on a different node. Thats why we use Cluster as externalTrafficPolicy. Here is the flow:

![traffic-without-snat](/personal-website/images/without_snat.png#center)

1. Client hits the Load Balancer and the LB will route the traffic to one of the nodes. 
2. On the node, kube-proxy handles the incoming traffic, modifes the destination in the packet -this is the DNAT - and forwards the packet to the other node. On this node kube-proxy forwards the traffic to the destination POD (on the picture above the connection arrow is direct between the first node and the POD, however the traffic goes through the kube-proxy of the second node as well.)
3. The target POD recieves the packet, and tries to answer for that. It sets the SRC as its IP and the destination will be the client. 
4. Because the packet is from the POD to the client it will be NAT-ed to the internet. However, this packet will be dropped. Why? Because the client sent a packet to the load balancer, it didnt send the packet to the VM or this POD. So, the TCP session is not able to complete. 

Let's see how SNAT-ting solves this problem. The figure is almost the same. The difference, when the packet reaches the first VM we do a second NAT, the SNAT. The source will be the IP of the node, the destination will be the pod. 

![traffic-with-snat](/personal-website/images/with_snat.png#center)

4. This time the TCP session is able to complete, because the source and destination matches.
5. The traffic goes back to the first node, where the node will unnat the packet twice. The result of this unNAT is the packet where the source is the LB and the DST is the client. 

The obvious drawback of the SNAT is that the target POD is not able to see the client's IP address thats why we want to use sometimes the Local settings.

#### externalTrafficPolicy: Local

In this case, kube-proxy will proxies the traffic only to pods that exist on the same node (local) as opposed to every pod regardless of where it was placed (Cluster). It ensures that the target POD will be able to see the client's IP address (because we dont have to use SNAT) and will be no extra network hops. 

The obvious drawback using the “Local” external traffic policy, is that traffic to our application may be imbalanced. See this on the following picture.

![traffic-imbalanced](/personal-website/images/imbalance.png#center)

The Cloud Loadbalance is only aware of Nodes, not PODs so it wont know how many PODs are on the given nodes. The cloud LB will route the traffic between the nodes equally. 

If our workload has a much more pods, and those PODs are distributed on different nodes, the imbalance problem will be less meaningful. See below:

![traffic-imbalanced](/personal-website/images/balanced.png#center)

## Conclusion
If you want to reduce the network hops you should definitely go with the Local option. Furthermore, most cases due to security reasons you want to be aware of the client's IPs. As for my experiences, I am using mostly the Local option, because most of my ingresses has the `nginx.ingress.kubernetes.io/whitelist-source-range` annotation. 