


Container networking	

Why should you learn container networking?

Working with containers and a container orchestrator like Kubernetes understanding the way containers communicate and send packages will make your life easier in many ways. I was motivated to learn so I could debug and diagnose problems in deployments. Kubernetes and it’s network solutions do a great job of abstracting away many of the mechanisms that make containers so special. Docker also abstracts away this functionality with it’s easy to use commands, but when something goes wrong it can be incredibly frustrating to figure out. 

Container networking is based on Linux networking and insights learned in either can be applied to both so you can use these skills on Linux machines to create a ‘real’ network. Learning this new skill you will also be able to create a network plugin for Kubernetes or at the very least feel more comfortable working with containers. Many of the concepts here are meant for beginners and for those with prior knowledge please use as a small refresher.

In this blog post you will first interact with the Linux kernel to become familiar with the commands, then you will create a virtual network. 


Then, you will use docker to understand docker’s use of the Linux kernel. 

To have a good base to build upon we need to know what a container and network are. 

A network refers to a group of entities that are uniquely addressable that can communicate with each other. This could be either an individual container, a machine, or some other device like a router.

A container can be considered synonymous with a Linux network namespace. Each container runtime might utilize a namespace differently. For example, with Docker each container gets its own namespace and in rckt a group of containers shares a namespace and is called a pod.

To avoid wasting any time we can jump in and explain as we go along
---
If you do not have access to a Linux operating system you can install multipass to easily spin up a virtual machine. 

To install using Homebrew:
```
brew cask install multipass
```

Spin up a vm with the name ‘linux-vm’ 
```
multipass launch -n linux-vm
```
To list the virtual machines in mulitpass use the command `multipass list`.

To enter into a shell in your vm use the following:
```
multipass shell linux-vm
```
---

To see the interfaces on your device enter
```
ip link list
```
This should print out devices that are are available.  Any networking device which has a driver loaded can be classified as an available device, in the output we see two devices `lo` and `enp0s2`. The `ip link` command will also output two lines for each device the link status and characteristics.


A network namespace is another copy of the network stack, with its own routes, firewall rules, and network devices. A process inherits its network namespace from its parent by default.	

Let’s create a two network namespaces `pb` and `jelly`
```
sudo ip netns add pb
sudo ip netns add jelly
```

Once they are added you can view them with `ip netns list`


You now have two namespaces that we can connect using a veth. The veth device is a virtual Ethernet device and you can think of it as a cable connecting two devices.  They can act as tunnels between network namespaces to create a bridge to a physical network device in another namespace, but can also be used as standalone network devices. 

Veth devices are always created in interconnected pairs. In veth pairs packets transmitted on one device in the pair are immediately received on the other device.  When either device is down the link state of the pair is down.

We are creating two veth pairs, the jelly namespace will connect to the veth-jelly end of the cable, and the other end of the cable will connect to a bridge that will create the network for our namespaces. We create the same cable and connect it to the bridge on the peanutbutter side.

To create both veth pair use the command:
```
sudo ip link add veth-pb type veth peer name bread-pb-veth
sudo ip link add veth-jelly type veth peer name bread-j-veth
```
Taking a look at the devices you will see the pair veth-bread1 and veth-bread2 on the host network.
```
ip link list
```
Now, that we have a veth pair in the host namespace, lets move the cable out into the jelly and pb namespace. 
```
sudo ip link set veth-pb netns pb
sudo ip link set veth-jelly netns jelly
```
If you check `ip link list` you will no longer find `veth-pb` and `veth-jelly` since they aren’t in the host namespace.

To see the ends of the cable you created, run the `ip link` command within the namespaces.
```
sudo ip netns exec jelly ip link
sudo ip netns exec pb ip link
```

You keep using `ip` commands and at this point you might be wondering, where the IPV4 addresses are? We have to retrace our steps a little bit so it makes sense. We began by creating two namespaces, then a virtual ethernet cable, and connected the devices to the cable. We have done all of this virtually and if we did this with hardware we would just connect the two devices with a cable so during those steps we would not yet deal with ip addresses. 

To create an ip address for the jelly namespace with the `ip address add` command for the device jelly:
```
sudo ip netns exec jelly \
ip address add 192.168.0.55/24 dev veth-jelly

sudo ip netns exec jelly \
ip link set veth-jelly up
```

Similarly, assign an address to pb as well 
```
sudo ip netns exec pb \
ip address add 192.168.0.56/24 dev veth-pb

sudo ip netns exec pb \
ip link set veth-pb up
```
Check the interfaces in the namespaces to check if they are correct with the next command. You are looking for the interfaces that you created `veth-pb` and `veth-jelly`. Make sure they have the ip addresses that you set. 
```
sudo ip netns exec jelly \
ip addr
sudo ip netns exec pb \
ip addr
```


Linux Bridge

```
sudo apt install bridge-utils
```
You now have both IPs and interfaces set, but we can’t establish communication with them.

That’s because there’s no interface in the default namespace that can send the traffic to those namespaces - we didn’t either configure addresses to the other side of the veth pairs or configured a bridge device.

With the creation of the bridge device, we’re then able to provide the necessary routing, properly forming the network:

Create your bridge, I called mine brd1 for bread that is needed for the sandwich
```
sudo ip link add name brd1 type bridge
# set the interface
sudo ip link set brd1 up
```
To check if the device was created `ip link`

With your bridge device set it's time to connect the bridge side of the veth cable to the bridge
```
sudo ip link set bread-pb-veth up
sudo ip link set bread-j-veth up
```

You can add the bread veth interfaces to the bridge by setting the bridge device as their master.
```
sudo ip link set bread-pb-veth master brd1
sudo ip link set bread-j-veth master brd1
```
To verify
```
bridge link show brd1
```

Try pinging  pb’s ip 192.168.0.56 from the jelly namespace
```
sudo ip netns exec jelly ping 192.168.0.56
```


---
Next blog
Docker

If you are missing docker, you can install it with the following

```
sudo snap install docker
```





In Kubernetes:

Cross node pod to pod communication

Any pod must be able to communicate to other pods in the cluster without network address translation

Any node must be able to communicate to any pod in the cluster without network address translation 

Networking is complex and abstracting away the networking functionality away from Kubernetes allows them to evolve separately from Kubernetes. At the same timeK8s need to able to consume the networking solutions

CNI = Container Network Interface provides a specification to add/connect containers and delete/disconnecting containers.

CNI is necessary to prevent overlap so work isn’t duplicated. 
CNI also makes sense to have networking standards to make sure everything works in conjunction.

3 parts to the CNI
Specification for connecting container runtimes and network solution
Code library - to build a CNI plugin
Command-line tool to execute a CNI plugin

CNI plugins are developing solutions to this problem differently and you can use more than one at a time. One plugin could be routing issuing IP addresses and routing traffic to make sure the right connection ends up at the desired pod, the other plugin could be enforcing networking security policies that prevent unwanted traffic reaching protected destinations. 

Some examples:
NSX-T
Calico
Flannel
Weave
Cilium
Canal
Kubernetes doesn’t care how networking is implemented in a cluster it only cares if pods can communicate with each other.

CNI abstracts away the networking functionality through a common standard that can be fulfilled through plugins that can provide the networking for your Kubernetes cluster

To have a foundation to build upon let’s use the CNI definition of a container and a network:

A container can be considered synonymous with a Linux network namespace. What unit this corresponds to depends on a particular container runtime implementation: for example, in implementations of the App Container Spec like rkt, each pod runs in a unique network namespace. In Docker, on the other hand, network namespaces generally exist for each separate Docker container.

A network refers to a group of entities that are uniquely addressable that can communicate with each other. This could be either an individual container (as specified above), a machine, or some other network device (e.g. a router). Containers can be conceptually added to or removed from one or more networks.


Sources
CNI from scratch
https://www.youtube.com/watch?v=zmYxdtFzK6s > https://github.com/eranyanay/cni-from-scratch

CNI Spec
https://github.com/containernetworking/cni/blob/master/SPEC.md 

Kubeacademy - intro to CNI
https://kube.academy/lessons/an-introduction-to-cni

Intro to container networking Rancher
https://rancher.com/learning-paths/introduction-to-container-networking/

Kind inside k8s cluster
https://d2iq.com/blog/running-kind-inside-a-kubernetes-cluster-for-continuous-integration

https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/

https://www.opencloudblog.com/?p=66

https://www.man7.org/linux/man-pages
