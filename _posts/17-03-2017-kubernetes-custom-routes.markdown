# Addin custom routes to Kubernetes on AWS

## Introduction

TODO
- VPN server as Pod
- HTTP-Proxy as Pod
- We want to connect without having to make HTTP-Proxy a VPN-Client
- Need to push route table entry for the VPN-Subnet to the VPN-Server (as we would in a traditional company or home setup)

## Solution
For finding our way to the problem's solution, we have a to look into how the single components and networking layers work. 

Our cluster is deployed using the AWS-Adapter of Kube-Up, so we have a relatively standard way of deploying:
- The cluster is encapsulated in its own VPC. Each node has its own subnet for the containers it is running. 
  These subnets resembled in routing table entries on the VPC's route tables.
  
 - Kubelet is running as a linux service, all other central Kubernetes components like kube-controller, kube-scheduler, etcd and kube-proxy are deployed as containers
 
 To get a better idea how we can add our custom route in the most stable way, let's look at how the networking layers interact with each other to find a proper entrypoint.
 
 ### Node container network and VPC
 Let's begin at the most basic level. How do the containers (regardless of Pods, services etc) expose their connections to the outside and how do we break node boundaries in our AWS setup.
 Especially for the second part of this question the answer varies on the cluster networking solution you are using (e.g. Flannel has other configurations than VPC), but the basic principles should be the same (or at least similar).
 
 On a node level, all containers started within the cluster get attached to a bridge docker net (in our case called cbr0). Ifconfig for the interfaces shows us an interface which is configured something like this:
 inet addr:10.244.1.1  Bcast:0.0.0.0  Mask:255.255.255.0
 
 and route -n shows us a matching routing entry so that the containers inside this net are available to the outer world:
 10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cbr0
 
 The CIDR of these subnets is unique on every node.
 
 So, what does this mean for communication to containers from outside of the nodes?
 - Containers are running inside their own subnet on a bridge interface of the host. Just as in your standard docker installation.
 - However, our K8s Node has additional routing information that tells it to forward every packet it receives for the docker subnet to the bridge network. By this every host can communicate as long as it sends the packets to the correct node.

If a node does not have a specific route configured to a host, it will try to access it over the default gateway. route -n also tells us about this:
0.0.0.0         172.20.0.1      0.0.0.0         UG    0      0        0 eth0

This means we are acesing 172.20.0.1 which is the gateway of the Kubernetes VPC if we don't have any other routing information. This means all calls to container IPs not in the own docker subnet, it will be propagated into the VPC.

Now, to make the containers accessible inside the whole cluster, K8s has to repeat setting the subnet routes just one level higher in our network topology. Enter VPC routing tables.

If we go to the AWS VPC management console, under Route-Tables, we will see at least one route table that is connected to kubernetes-vpc. In the "Routes" tab, we will see something like this:

Destination   | Target                      | Status | Propagated
172.20.0.0/16 | local                       | Active | No
0.0.0.0/0     |	igw-redacted	              | Active | No
10.244.0.0/24 | eni-redacted1 / i-redacted1 | Active | No
10.244.1.0/24 | eni-redacted2 / i-redacted2 | Active | No
10.246.0.0/24 | eni-redacted3 / i-redacted3 | Active | No

The eni-redated\* are instance IDs i-redacted\* are interface IDs. The implications of this table are the following:
- Everything that's addressed to 172.20.0.0/16 (our kubernetes VPC net) gets forwarded within the VPC
- Everything that isn't caught by any other rule is forwarded to the VPCs internet gateway and goes outside of the VPC
- Everything that's addressed to 10.244.0.0/24, 10.244.1.0/24 and 10.246.0.0/24 (do these look familiar?) is routed to a specific host which are the masters and nodes in our Kubernetes cluster.

With this configuration, we can route packages from every container in the cluster to every other. However, we still need to know the exact IP of the container and since this tends to change a lot (and I don't want to get paged in the middle of the night just to update one stupid little IP in the routing table), let's keep on digging. Fortunately, K8s comes with a component that helps us decouple the reference of a given SET OF CONTAINERS from their actual assigned address...

### Kubernetes Services

