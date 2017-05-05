# Adding custom routes to Kubernetes on AWS

At my day job, I am running an IoT cloud setup which includes connecting devices located on customer sites. Some features require initiating a connection to the device from within the cloud services. To do this in a secure way, we are connecting the devices over VPN. For this, we have deployed an OpenVPN server as a Pod in Kubernetes. The component which acts as a gateway between clients and devices is basically an HTTP-Proxy with some additional domain specific behavior, so it needs to be able to forward incoming HTTP calls to the device over VPN. This means that within our Kubernetes cluster we need to configure a route on IP level that lets the Proxy Pod route packets to the VPN net using the VPN Server Pod.

One idea to implement this would be to just make the Proxy component a VPN client. However, since we already have a connection between the services via the Kubernetes cluster network, this feels kind of wrong. We are adding unneccessary overhead to the services and that could bite us in the future. Also it's just bad practice to use VPN (or some other remote connection protocol) where it isn't needed and managing the VPN client configurations adds complexity when scaling the Proxy pods horizontally. The cleaner solution would be to add the needed route on cluster network level to forward packets destined to the VPN accordingly.

## Functional Base
For finding our way to the problem's solution, we have to look at how the single components and networking layers work. 

Our cluster is deployed using the AWS-Adapter of kube-up, so we have a pretty standard deployment:
- The cluster is encapsulated in its own VPC. Each node has its own subnet for the containers it is running. 
  These subnets are connected via routing table entries on the VPC's route tables.
  
 - Kubelet is running as a linux service, all other central Kubernetes components like kube-controller, kube-scheduler, etcd and kube-proxy are deployed as containers.
 
Let's look at how the single Kubernetes components interact on a networking point of view.
 
### Node container network and VPC
This is the least abstract level of our Pods. How do the actual containers running on the nodes expose their connections to the outside? And how are packets routed between Pods on different nodes?
 
Especially for the question the answer may vary based on the cluster networking solution you are using (e.g. Flannel has other configurations than VPC), but the basic principles should be the same (or at least similar).
 
On a node level, all containers started within the cluster get attached to a docker bridge network (cbr0). `Ifconfig` shows us an interface which is configured something like this:
```
inet addr:10.244.1.1  Bcast:0.0.0.0  Mask:255.255.255.0
```
Each node in the cluster has a different CIDR for cbr0. `route -n` shows us a matching routing entry so that the containers inside the net are available to the outer world: 
```
10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cbr0
```
 
Now, what does this mean for communication to containers from outside of the nodes?
 - Containers are running inside their own subnet on a bridge interface of the host, isolated from non-container components,  just as in a standard docker installation.
 - Our K8s Node has additional routing information that tells it to forward every packet it receives for the docker subnet to the bridge network. By this, every host can communicate with the containers as long as it sends the packets to the correct node.

If a node does not have a specific route configured to a host, it will try to access it over the default gateway. `route -n` also tells us about this:
```
0.0.0.0         172.20.0.1      0.0.0.0         UG    0      0        0 eth0 
```

This means that if we don't have any other routing information, we are routing to 172.20.0.1, which is the gateway of the VPC.
With this configuration on each node, all calls to Containers that have to cross node boundaries will be forwarded to the VPC. K8s (or more precisely kube-up) still has to do is tell the default gateway which subnet is connected to which node so it routes the packages correctly. Basically, this is the same configuration as above, but now on VPC level instead of the nodes.

If we go to the AWS VPC management console, under Route-Tables we will see at least one route table that is connected to kubernetes-vpc. In the "Routes" tab, we will see something like this:

|Destination   | Target                      | Status | Propagated |
|--------------|:---------------------------:|:------:|-----------:|
|172.20.0.0/16 | local                       | Active | No         |
|0.0.0.0/0     | igw-redacted	               | Active | No         |
|10.244.0.0/24 | eni-redacted1 / i-redacted1 | Active | No         |
|10.244.1.0/24 | eni-redacted2 / i-redacted2 | Active | No         |
|10.246.0.0/24 | eni-redacted3 / i-redacted3 | Active | No         |

The eni-redacted\* entries are instance IDs, i-redacted\* entries are interface IDs. This table configures the following behavior:
- Everything that's addressed to 172.20.0.0/16 (our kubernetes VPC net) gets forwarded within the VPC
- Everything that isn't caught by any other rule is forwarded to the VPCs internet gateway and goes outside of the VPC
- Everything that's addressed to 10.244.0.0/24, 10.244.1.0/24 and 10.246.0.0/24 (the node specific Docker subnets) is routed to a specific host which are the masters and nodes in our cluster.

With this configuration, we can route packages from every container in the cluster to every other, no matter of the node they are located on. However, we still need to know the exact IP of the container we want to route to. Since Pods and their containers tend to be destroyed and recreated and thus get assigned new IPs frequently (and I don't want to get paged in the middle of the night just to update one IP in the routing table after a Pod restart), let's keep on digging to find a more solid base. 

Fortunately, K8s comes with a component that helps us decouple the reference of a given set of containers from their actual address on the docker subnet...

### Kubernetes services
Kubernetes services are nice components for cutting hard references between Pods and making them accessible from the outside. Speaking in more general terms, a service is a reverse proxy/load balancer. Instead of accessing a container of a Pod directly, a host calls the service which picks one pod from the reference it holds and forwards the request. By this, we can create and delete pods without having to change references to them all the time, we just use the DNS of a service that is fairly stable. Horizontally scaling components by running several instances would also be very hard if we didn't have services (or loadbalancers in general). Let's look into how they are integrated into the cluster network to see how we can leverage services for our custom route.

The networking for services happens on each Node and is done by kube-proxy. Kube-Proxy has two modes: IPTables where it will render service routings into IPTables rules so each call to a service will get routed on a node level automatically or userspace mode where it intercepts calls and actively forwards calls to services. In our case we are using IPTables mode.

For knowing when a service has been created, modified or deleted, kube-proxy polls the API server for change events and holds an internal service map with all service information to determine what to change in order to render the service correctly to routing rules. 

The services are rendered as NAT rules on the nodes, so `iptables -t nat -L` will show us what's configured exactly. We will get a lot of rules that resemble all the services in the cluster, but in essence this example shows what's going on:
``` 
Chain KUBE-MARK-MASQ (58 references)  
target     prot opt source               destination
MARK       all  --  anywhere             anywhere             MARK or 0x4000
```
```
Chain KUBE-SEP-7ZAB4MEUNQGISTVW (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  aws-ip.aws-region.internal  anywhere             /* kube-system/kubernetes-dashboard: */
DNAT       tcp  --  anywhere             anywhere             /* kube-system/kubernetes-dashboard: */ tcp to:10.244.1.11:9090
```

``` 
Chain KUBE-SERVICES (2 references)
target     prot opt source               destination
KUBE-MARK-MASQ  tcp  -- !pod-hosting-node-ip.pod-hosting-node-region.internal/16  ip-10-0-91-181.aws-region.compute.internal  /* kube-system/kubernetes-dashboard: cluster IP */ tcp dpt:http
KUBE-SVC-XGLOHA7QRQ3V22RZ  tcp  --  anywhere             ip-10-0-91-181.aws-region.compute.internal  /* kube-system/kubernetes-dashboard: cluster IP */ tcp dpt:http
```

```
Chain KUBE-SVC-XGLOHA7QRQ3V22RZ (1 references)
target     prot opt source               destination
KUBE-SEP-7ZAB4MEUNQGISTVW  all  --  anywhere             anywhere             /* kube-system/kubernetes-dashboard: */
```

There are also a some more general roules that handle prerouting, postrouting and general access to the Docker and Kubernetes nets but since I'm no expert for NAT or networking, I'll spare us the embarrassment of doing a deep dive. The three chains above are the ones that do the work of mapping the internal clusterIP of the service (in this case 10.0.91.181) to the Kubernetes net IP of the Pod (in this case 10.244.1.11) for the kubernetes-dashboard service. The third one determines if we have to route internally or externally, the fourth one can be used for further access controll (I guess), the second one does the actual mapping to the Pod(s) and the first one masks the calls.

## Concept for a custom route mechanism
For our routing to the VPN IPs, we want to add similar rules for the VPN net that point to the VPN Server pod.

The probably least intrusive way of adding our desired route to the cluster network is building an own component that combines the basic mechanisms described above and adds addional routes accordingly. The approach we take for this is the following:

1. We will deploy and own component as a Daemon-Set so it runs on each node
2. We will be able to suply a list of subnets and the Kubernetes-Servicenames they should be routed to as thirdparty resources
3. Our new component will watch the Kubernetes API for changes on these resources and Endpoints to the configured services and keep the routing table entries up to date on a node level. 

This allows us to very precisely control which routes will be added and makes us independent from any other components. Since we will be making the routing on a node level, the same way we are doing the kubernetes container net to the outer world, we will be independent from infrastructure and cluster networking solutions, so we should be able to use this in a broad number of different setups.

I haven't started implementation yet but plan to start with it soon. The sourcecode for this is located at https://github.com/oxygen0211/kubernetes-net-router. Contributions are very welcome.

## UPDATE
After working a bit more on this and talking to other Kubernetes users who have a bit more experience than me, I came to the conclusion that running an OpenVPN server inside the Kubernetes cluster is too much state. In the past we had some of the highly dymanic features of Kubernetes, like dynamic scaling of both nodes and pods, blocked because we alway had this OpenVPN server pod which has a lot of long-living sessions sticking to it and should not be rescheduled by any means since it would always cause our customer devices to loose connectivity for a while. Instead, I went for running the Server on an own VM (but still encapsulated in a Docker container) which is not part of the K8s-cluster but is in the same VPC and Subnet, just like we do with other stateful applications such as databases, block- or object stores. I recommend to everyone working with Kubernetes and similar projects to follow the general rule that every state that needs to survive a pod- (or, to speak in more general terms, a container-) restart should not be stored within the K8s-cluster.

This means we don't need to develop and maintain another component but we still have to set up routing correctly. This can be done with the mechanisms described above.

### Routing to the Openvpn-Server VM
First of all, we need to enable the machines within the VPC - especially our K8s-Nodes that still run our proxy and all other stateless services - to route calls to VPN connected hosts to our OpenVPN server machine. Since on machine level we have already a catch-all that forwards calls to any unknown host to the VPC, we just need to configure a route on VPC level. This is done in the VPC management console in route tables. Just add a route with your VPN CIDR (e.g. "10.8.0.0/16") and target it to the eth0 interface of your VPN Server VM (this is found in EC2 management console in the instance details).

### Routing to the Docker container
We are still running our VPN Server as Docker container. This i mainly to save time and nerves on different ends. First, we have this already running in production, so by reusing the container, we don't have the risk of accidentally configuring something that breaks client connectivity - or at least we can be more secure what change broke compatibility if we do something. Second, we have it easier to do modifications and testing on the server before updating it in production, just update the container config, rebuild, test it locally, in stating/preproduction, etc. and then update the Server, maybe even in a rolling update way - more on this in an own post soon.

However, this means that we need to configure our server to cross interface boundaries internally. If we look at our Server VM's routes with ```route -n```, we see something like this:

```
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.20.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-<random string>
172.20.0.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

What we want to do now is to add a new route that will forward calls to our VPN net to the VPN Docker container. By utilizing ```docker inspect <container name or ID>```, you can find that one out and by calling ```route add -net <VPN-Net IP, e.g 10.8.0.0> netmask <VPN-Net mask, e.g. 255.255.0.0> gw <container IP>```, we will add routing of the calls to the container. Not that I conciously didn't specify a device on ```route add```, this is to let route choose the correct one on its own. We should now be able to ping the VPN server on its VPN IP, in that case we can expect it to 10.8.0.1, if that doesn't work, check the IP with ```ifconfig``` inside the container first. Of course, this should be persistent over machine reboots (we hope this doesn't happen too often for the reasons state above, but we don't want to have additional trouble with missing routes once it happened), so we need to add this to the interface configuration. Since our VPN server VM is running on Ubuntu 16.04, this will be done by adding ```post-up route add -net <VPN-Net IP, e.g 10.8.0.0> netmask <VPN-Net mask, e.g. 255.255.0.0> gw <container IP>``` to our eth0 interface definition in /etc/network/interfaces.d/50-cloud-init.cfg. 

If our container is not running on the host net (which has caused the VPN net to not work properly anymore in my tries), Docker will block all communication to the container except for the forwarded ports. To overcome this, we have to add additinoal rules to iptables ``` sudo iptables -A DOCKER -j ACCEPT -p all -d <container hostname or IP, check with already existing rules for forwarded ports> ```

You see, we have some hard coding and manual steps here, but for now, this is OK since we explictly want this VM and container to be long running and do not expect it to set itself up fully automatically. We also have to provision our VM somehow, so this is just another step following Docker installation, adding the VPN Server configuration, starting the container, etc. In our case, there's an Ansible playbook handling this (closed source, sorry) and I would recommend anyone to do automate such infrastructure deployments some how (#infrastructureascode #DevOps) with Puppet, Check, Ansible or similar or creating your own VM image if you prefer that. Once your machine gets terminated or crashes beyond repair otherwise, you will be thankful for it. 

### Routing to VPN clients inside the container
OK, back to topic. As you see, by now we can ping our server but no connected host. This is because there's still a route missing inside our container, so we will update our VPN server container to do this.

