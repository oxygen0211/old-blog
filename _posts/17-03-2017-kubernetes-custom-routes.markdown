# Adding custom routes to Kubernetes on AWS

At my day job, I am running in IoT cloud setup which includes connecting devices located on customer sites. Some features require initiating a connecting to the device from within the cloud services. To do this in a secure way, we are connecting the devices over VPN. For this, we have deployed an OpenVPN server as a Pod in Kubernetes. The component which acts as a gateway from a client to the devices is basically an HTTP-Proxy with some additional domain specific behavior, so it needs to be able to forward incoming HTTP calls to the device over VPN, which means that within our Kubernetes cluster we need to configure a route on IP level that lets the Proxy Pod route packets to the VPN net over the VPN Server pod.

One idea to implement this would be to just make the Proxy component a VPN client. However, since we already have a connection between the services via the Kubernetes cluster network, this kind of feels wrong. We are adding unneccessary overhead to the services and that could bite us in the future. Also it's just bad practice to use VPN (or some other remote connection protocol) where it isn't needed. The cleaner solution would be to add the needed route on cluster network level to forward packets destined to the VPN accordingly.

## Functional Base
For finding our way to the problem's solution, we have a to look into how the single components and networking layers work. 

Our cluster is deployed using the AWS-Adapter of kube-up, so we have a pretty standard deployment:
- The cluster is encapsulated in its own VPC. Each node has its own subnet for the containers it is running. 
  These subnets are connected via routing table entries on the VPC's route tables.
  
 - Kubelet is running as a linux service, all other central Kubernetes components like kube-controller, kube-scheduler, etcd and kube-proxy are deployed as containers.
 
 To get a better idea how we can add our custom route in the most stable way, let's look at how the single Kubernetes components interact on a networking point of view.
 
 ### Node container network and VPC
 Let's begin at the least abstract level. How do the actual containers running on the nodes expose their connections to the outside and how do we break node boundaries in our AWS setup?
 
 Especially for the second part of this question the answer varies on the cluster networking solution you are using (e.g. Flannel has other configurations than VPC), but the basic principles should be the same (or at least similar).
 
 On a node level, all containers started within the cluster get attached to a bridge docker net (cbr0). `Ifconfig` for the interfaces shows us an interface which is configured something like this:
`inet addr:10.244.1.1  Bcast:0.0.0.0  Mask:255.255.255.0
 
 and `route -n` shows us a matching routing entry so that the containers inside this net are available to the outer world: `10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cbr0`
 
 The CIDR of these subnets is unique on every node.
 
 So, what does this mean for communication to containers from outside of the nodes?
 - Containers are running inside their own subnet on a bridge interface of the host. Just as in your standard docker installation.
 - However, our K8s Node has additional routing information that tells it to forward every packet it receives for the docker subnet to the bridge network. By this every host can communicate with the containers as long as it sends the packets to the correct node.

If a node does not have a specific route configured to a host, it will try to access it over the default gateway. `route -n` also tells us about this:
`0.0.0.0         172.20.0.1      0.0.0.0         UG    0      0        0 eth0`

This means that if we don't have any other routing information, we are routing to 172.20.0.1, which is the gateway of the Kubernetes VPC, so for all calls to container IPs not in the own docker subnet of a node will be routed into the VPC.

With this configuration on each node, all calls to Containers that have to cross node boundaries forwarded to the VPC. All K8s has to do to else is tell the default gateway which subnet is connected to which node so it routes the packages correctly. Basically, this is the same configuration as above, but now on VPC level instead of on the linux hosts.

If we go to the AWS VPC management console, under Route-Tables we will see at least one route table that is connected to kubernetes-vpc. In the "Routes" tab, we will see something like this:

|Destination   | Target                      | Status | Propagated |
|--------------|:---------------------------:|:------:|-----------:|
|172.20.0.0/16 | local                       | Active | No         |
|0.0.0.0/0     |	igw-redacted	             | Active | No        |
|10.244.0.0/24 | eni-redacted1 / i-redacted1 | Active | No         |
|10.244.1.0/24 | eni-redacted2 / i-redacted2 | Active | No         |
|10.246.0.0/24 | eni-redacted3 / i-redacted3 | Active | No         |

The eni-redated\* entries are instance IDs, i-redacted\* entries are interface IDs. The implications of this table are the following:
- Everything that's addressed to 172.20.0.0/16 (our kubernetes VPC net) gets forwarded within the VPC
- Everything that isn't caught by any other rule is forwarded to the VPCs internet gateway and goes outside of the VPC
- Everything that's addressed to 10.244.0.0/24, 10.244.1.0/24 and 10.246.0.0/24 (the node specific Docker subnets) is routed to a specific host which are the masters and nodes in our Kubernetes cluster.

With this configuration, we can route packages from every container in the cluster to every other. However, we still need to know the exact IP of the container we want to route to. Since Pods and their containers tend to often be destroyed and recreated (and I don't want to get paged in the middle of the night just to update one stupid little IP in the routing table), let's keep on digging to find a more solid base. Fortunately, K8s comes with a component that helps us decouple the reference of a given SET OF CONTAINERS from their actual assigned address...

### Kubernetes services
Kubernetes services are nice components for cutting hard references between Pods and - by this - Docker containers. Speaking in more general terms, a service is a reverse proxy/load balancer. Instead of accessing a container of a Pod directly, a host calls the service which holds reference to one or more containers, picks one and forwards the request. By this, we can create and delete pods up and down without having to change the references all the time. Horizontally scaling components by running several instances would also be very hard if we didn't have services (or loadbalancers in general). Let's dig into how they are integrated into the cluster network to see how we can leverage services for our custom route.

The networking for services happens on each Node in kube-proxy. Kube-Proxy has two modes: IPTables where it will render service routings into IPTables rules so each call to a service will get routed on a node level automatically or userspace mode where it intercepts calls and actively forwards calls to services. In our case we are using the IPTables version, so we'll have a closer look into this.

For knowing when a service has been created, modified or deleted, it polls the API server for change events and holds an internal service map with all service information to determine what to change in order to render the service correctly to routing rules. 

The services are rendered as NAT rules on the nodes, so `iptables -t nat -L` will show us what's under the hood. We will get a whole lot of rules that resemble all the services in the cluster, but in essence this example shows what's going on:
`Chain KUBE-MARK-MASQ (58 references)
target     prot opt source               destination
MARK       all  --  anywhere             anywhere             MARK or 0x4000`

`Chain KUBE-SEP-7ZAB4MEUNQGISTVW (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  aws-ip.aws-region.internal  anywhere             /* kube-system/kubernetes-dashboard: */
DNAT       tcp  --  anywhere             anywhere             /* kube-system/kubernetes-dashboard: */ tcp to:10.244.1.11:9090`

`Chain KUBE-SERVICES (2 references)
target     prot opt source               destination
KUBE-MARK-MASQ  tcp  -- !pod-hosting-node-ip.pod-hosting-node-region.internal/16  ip-10-0-91-181.aws-region.compute.internal  /* kube-system/kubernetes-dashboard: cluster IP */ tcp dpt:http
KUBE-SVC-XGLOHA7QRQ3V22RZ  tcp  --  anywhere             ip-10-0-91-181.aws-region.compute.internal  /* kube-system/kubernetes-dashboard: cluster IP */ tcp dpt:http`

`Chain KUBE-SVC-XGLOHA7QRQ3V22RZ (1 references)
target     prot opt source               destination
KUBE-SEP-7ZAB4MEUNQGISTVW  all  --  anywhere             anywhere             /* kube-system/kubernetes-dashboard: */`

There are alo a some more general roules that handle prerouting, postrouting and general access to the Docker and Kubernetes nets but since I'm no expert for NAT or networking, I spare us the embarrassment of a deep dive. Basically, the three chains above are the ones that do the work of mapping the internal clusterIP of the service (in this case 10.0.91.181) to the Kubernetes net IP of the Pod (in this case 10.244.1.11) for the kubernetes-dashboard service. The third one determines if we have to route internally or externally, the fourth one can be used for further access controll (I guess), the second one does the actual mapping to the Pod(s) and the first one masks internal calls.

For our routing to the VPN IPs, we want to add similar rules for the VPN net that point to the VPN Server pod. We just have to find an elegant way to implement this...

##Implementing custom route mechanism
The probably least intrusive way of adding our desired route to the cluster network is building an own component that combines the basic mechanisms described above and adds addional routes accordingly. The approach we take for this is the following:

1. We will deploy and own compnent as a Daemon-Set so it runs on each node
2. In a config file, we will be able to suply a list of subnets and the Kubernetes-Servicenames they should be routed to
3. Our new component will periodically poll the Kubernetes API for Endpoint changes to the configured services and keep the routing table entries up to date on a node level. 

This approach allows us to very precisely control which routes will be added and makes us independent from any other components. Since we will be making the routing on a node level, the same way we are doing the kubernetes container net to the outer world, we will be independent from infrastructure and cluster networking solutions, so we should be able to use this in a broad number of different setups.

The implementation for this is located at https://github.com/oxygen0211/kubernetes-net-router. Contributions are very welcome.
