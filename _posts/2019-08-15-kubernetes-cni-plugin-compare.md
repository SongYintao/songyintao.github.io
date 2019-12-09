---
title: Kubernetes CNI plugin compare
layout: post
tags: [kubernetes,cni]
---

[经典文章转载](https://rancher.com/blog/2019/2019-03-21-comparing-kubernetes-cni-providers-flannel-calico-canal-and-weave/)

# Comparing Kubernetes CNI Providers: Flannel, Calico, Canal, and Weave

## Introduction

Network architecture is one of the more complicated aspects of many Kubernetes installations. The Kubernetes networking model itself demands [certain network features](https://kubernetes.io/docs/concepts/cluster-administration/networking/#the-kubernetes-network-model) but allows for some flexibility regarding the implementation. As a result, various projects have been released to address specific environments and requirements.

In this article, we’ll explore the most popular **CNI plugins**: flannel, calico, weave, and canal (technically a combination of multiple plugins). CNI stands for [container network interface](https://github.com/containernetworking/cni), a standard designed to make it easy to configure container networking when containers are created or destroyed. These plugins do the work of making sure that Kubernetes’ networking requirements are satisfied and providing the networking features that cluster administrators require.

## Background

**Container networking** is the mechanism through which containers can optionally connect to other containers, the host, and outside networks like the internet. Container runtimes offer various networking modes, each of which results in a different experience. For example [Docker](https://www.docker.com/) can configure the following networks for a container by default:

- **none**: Adds the container to a container-specific network stack with no connectivity.
- **host**: Adds the container to the host machine’s network stack, with no isolation.
- **default bridge**: The default networking mode. Each container can connect with one another by IP address.
- **custom bridge**: User-defined bridge networks with additional flexibility, isolation, and convenience features.

Docker also allows you to configure more advanced networking, including multi-host overlay networking, with additional drivers and plugins.

The idea behind the CNI initiative is to create a framework for dynamically configuring the appropriate network configuration and resources when containers are provisioned or destroyed. The [CNI spec](https://github.com/containernetworking/cni/blob/master/SPEC.md)outlines a plugin interface for container runtimes to coordinate with plugins to configure networking.

Plugins are responsible for provisioning and managing an IP address to the interface and usually provide functionality related to IP management, IP-per-container assignment, and multi-host connectivity. The container runtime calls the networking plugins to allocate IP addresses and configure networking when the container starts and calls it again when the container is deleted to clean up those resources.

The runtime or orchestrator decides on the network a container should join and the plugin that it needs to call. The plugin then adds the interface into the container network namespace as one side of a `veth` pair. It then makes changes on the host machine, including wiring up the other part of the `veth` to a network bridge. Afterwards, it allocates an IP address and sets up routes by calling a separate IPAM (IP Address Management) plugin.

In the context of Kubernetes, this relationship allows `kubelet` to automatically configure networking for the pods it starts by calling the plugins it finds at appropriate times.

## Terminology

Before we compare take a look at the available CNI plugins, it’s helpful to go over some terminology that you might see while reading this or other sources discussion CNI.

Some of the most common terms include:

- **Layer 2 networking:** The “data link” layer of the OSI (Open Systems Interconnection) networking model. Layer 2 deals with delivery of frames between two adjacent nodes on a network. Ethernet is a noteworthy example of Layer 2 networking, with MAC represented as a sublayer.
- **Layer 3 networking:** The “network” layer of the OSI networking model. Layer 3’s primary concern involves routing packets between hosts on top of the layer 2 connections. IPv4, IPv6, and ICMP are examples of Layer 3 networking protocols.
- **VXLAN:** Stands for “virtual extensible LAN”. Primarily, VXLAN is used to help large cloud deployments scale by encapsulating layer 2 Ethernet frames within UDP datagrams. VXLAN virtualization is similar to VLAN, but offers more flexibility and power (VLANs were limited to only 4,096 network IDs). VXLAN is an encapsulation and overlay protocol that runs on top of existing networks.
- **Overlay network:** An overlay network is a virtual, logical network built on top of an existing network. Overlay networks are often used to provide useful abstractions on top of existing networks and to separate and secure different logical networks.
- **Encapsulation:** Encapsulation is the process of wrapping network packets in additional layer to provide additional context and information. In overlay networks, encapsulation is used to translate from the virtual network to the underlying address space to route to a different location (where the packet can be de-encapsulated and continue to its destination).
- **Mesh network:** A mesh network is one in which each node connects to many other nodes to cooperate on routing and achieve greater connectivity. Network meshes provide more reliable networking by allowing routing through multiple paths. The downside of a network mesh is that each additional node can add significant overhead.
- **BGP:** Stands for “border gateway protocol” and is used to manage how packets are routed between edge routers. BGP helps figure out how to send a packet from one network to another by taking into account available paths, routing rules, and specific network policies. BGP is sometimes used as the routing mechanism in CNI plugins instead of encapsulated overlay networks.

Now that we’ve introduced some of the technology that enables various plugins, we’re ready to explore some of the most popular CNI options.

## CNI Comparison

### Flannel

[Flannel](https://github.com/coreos/flannel), a project developed by the [CoreOS](https://coreos.com/), is perhaps the most straightforward and popular CNI plugin available. It is one of the most mature examples of networking fabric for container orchestration systems, intended to allow for better inter-container and inter-host networking. As the CNI concept took off, a [CNI plugin for Flannel](https://github.com/containernetworking/plugins/tree/master/plugins/meta/flannel) was an early entry.

Compared to some other options, Flannel is relatively easy to install and configure. It is packaged as a single binary called `flanneld` and can be installed by default by many common Kubernetes cluster deployment tools and in many Kubernetes distributions. Flannel can use the Kubernetes cluster’s existing `etcd` cluster to store its state information using the API to avoid having to provision a dedicated data store.

Flannel configures a layer 3 IPv4 overlay network. A large internal network is created that spans across every node within the cluster. Within this overlay network, each node is given a subnet to allocate IP addresses internally. As pods are provisioned, the Docker bridge interface on each node allocates an address for each new container. Pods within the same host can communicate using the Docker bridge, while pods on different hosts will have their traffic encapsulated in UDP packets by `flanneld` for routing to the appropriate destination.

Flannel has several different types of backends available for encapsulation and routing. The default and recommended approach is to use VXLAN, as it offers both good performance and is less manual intervention than other options.

Overall, Flannel is a good choice for most users. From an administrative perspective, it offers a simple networking model that sets up an environment that’s suitable for most use cases when you only need the basics. In general, it’s a safe bet to start out with Flannel until you need something that it cannot provide.

### Calico

[Project Calico](https://www.projectcalico.org/), or just Calico, is another popular networking option in the Kubernetes ecosystem. While Flannel is positioned as the simple choice, Calico is best known for its performance, flexibility, and power. Calico takes a more holistic view of networking, concerning itself not only with providing network connectivity between hosts and pods, but also with network security and administration. The [Calico CNI plugin](https://github.com/projectcalico/cni-plugin) wraps Calico functionality within the CNI framework.

On a freshly provisioned Kubernetes cluster that meets the [system requirements](https://docs.projectcalico.org/latest/getting-started/kubernetes/requirements), Calico can be deployed quickly by applying a single manifest file. If you are interested in Calico’s optional network policy capabilities, you can enable them by applying an additional manifest to your cluster.

Although the actions needed to deploy Calico seem fairly straightforward, the network environment it creates has both simple and complex attributes. Unlike Flannel, Calico does not use an overlay network. Instead, Calico configures a layer 3 network that uses the [BGP routing protocol](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) to route packets between hosts. This means that packets do not need to be wrapped in an extra layer of encapsulation when moving between hosts. The BGP routing mechanism can direct packets natively without an extra step of wrapping traffic in an additional layer of traffic.

Besides the performance that this offers, one side effect of this is that it allows for more conventional troubleshooting when network problems arise. While encapsulated solutions using technologies like VXLAN work well, the process manipulates packets in a way that can make tracing difficult. With Calico, the standard debugging tools have access to the same information they would in simple environments, making it easier for a wider range of developers and administrators to understand behavior.

In addition to networking connectivity, Calico is well-known for its advanced network features.[Network policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) is one of its most sought after capabilities. In addition, Calico can also integrate with [Istio](https://istio.io/), a service mesh, to interpret and enforce policy for workloads within the cluster both at the service mesh layer and the network infrastructure layer. This means that you can configure powerful rules describing how pods should be able to send and accept traffic, improving security and control over your networking environment.

Project Calico is a good choice for environments that support its requirements and when performance and features like network policy are important. Additionally, Calico offers commercial support if you’re seeking a support contract or want to keep that option open for the future. In general, it’s a good choice for when you want to be able to control your network instead of just configuring it once and forgetting about it.

### Canal

[Canal](https://github.com/projectcalico/canal) is an interesting option for quite a few reasons.

First of all, Canal *was* the name for a project that sought to integrate the networking layer provided by flannel with the networking policy capabilities of Calico. As the contributors worked through the details however, it became apparent that a full integration was not necessarily needed if work was done on both projects to ensure standardization and flexibility. As a result, the official project became somewhat defunct, but the intended ability to deploy the two technology together was achieved. For this reason, it’s still sometimes easiest to refer to the combination as “Canal” even if the project no longer exists.

Because Canal is a combination of Flannel and Calico, its benefits are also at the intersection of these two technologies. The networking layer is the simple overlay provided by Flannel that works across many different deployment environments without much additional configuration. The network policy capabilities layered on top supplement the base network with Calico’s powerful networking rule evaluation to provide additional security and control.

After ensuring that the cluster fulfills the necessary [system requirements](https://docs.projectcalico.org/latest/getting-started/kubernetes/requirements), Canal can be deployed by applying two manifests, making it no more difficult to configure than either of the projects on their own. Canal is a good way for teams to start to experiment and gain experience with network policy before they’re ready to experiment with changing their actual networking.

In general, Canal is a good choice if you like the networking model that Flannel provides but find some of Calico’s features enticing. The ability define network policy rules is a huge advantage from a security perspective and is, in many ways, Calico’s killer feature. Being able to apply that technology onto a familiar networking layer means that you can get a more capable environment without having to go through much of a transition.

### Weave Net

[Weave Net](https://www.weave.works/oss/net/) by [Weaveworks](https://www.weave.works/) is a CNI-capable networking option for Kubernetes that offers a different paradigm than the others we’ve discussed so far. Weave creates a mesh overlay network between each of the nodes in the cluster, allowing for flexible routing between participants. This, coupled with a few other unique features, allows Weave to intelligently route in situations that might otherwise cause problems.

To create its network, Weave relies on a routing component installed on each host in the network. These routers then exchange topology information to maintain an up-to-date view of the available network landscape. When looking to send traffic to a pod located on a different node, the weave router makes an automatic decision whether to send it via “fast datapath” or to fall back on the “sleeve” packet forwarding method.

Fast datapath is an approach that relies on the kernel’s native Open vSwitch datapath module to forward packets to the appropriate pod without moving in and out of userspace multiple times. The Weave router updates the Open vSwitch configuration to ensure that the kernel layer has accurate information about how to route incoming packets. In contrast, sleeve mode is available as a backup when the networking topology isn’t suitable for fast datapath routing. It is a slower encapsulation mode that can route packets in instances where fast datapath does not have the necessary routing information or connectivity. As traffic flows through the routers, they learn which peers are associated with which MAC addresses, allowing them to route more intelligently with fewer hops for subsequent traffic. This same mechanism helps each node self-correct when a network change alters the available routes.

Like Calico, Weave also provides network policy capabilities for your cluster. This is automatically installed and configured when you set up Weave, so no additional configuration is necessary beyond adding your network rules. One thing that Weave provides that the other options do not is easy encryption for the entire network. While it adds quite a bit of network overhead, Weave can be configured to automatically encrypt all routed traffic by using [NaCl encryption](http://nacl.cr.yp.to/) for sleeve traffic and, since it needs to encrypt VXLAN traffic in the kernel, [IPsec ESP](https://tools.ietf.org/html/rfc2406) for fast datapath traffic.

Weave is a great option for those looking for feature rich networking without adding a large amount of complexity or management. It is relatively easy to set up, offers many built-in and automatically configured features, and can provide routing in scenarios where other solutions might fail. The mesh topography does put a limit on the size of the network that can be reasonably accommodated, but for most users, this won’t be a problem. Additionally, Weave offers paid support for organizations that prefer to be able to have someone to contact for help and troubleshooting.

## Conclusion

Kubernetes’ adoption of the CNI standard allows for many different network solutions to exist within the same ecosystem. The diversity of options available means that most users will be able to find a CNI plugin that suits their current needs and deployment environment, while also providing solutions when their circumstances change. Operating requirements vary immensely between organizations, so having a number of mature solutions with different levels of complexity and feature richness helps Kubernetes satisfy unique requirements while still offering a fairly consistent user experience.

## More Resources on Kubernetes Networking

### Dive deeper into Kubernetes Networking eBook

For a more detailed guide into Kubernetes network architecture, check out our free ebook [“Diving Deep into Kubernetes Networking”](https://info.rancher.com/kubernetes-networking-deep-dive). This 42-page guide covers important networking topics thoroughly, including the Kubernetes networking model and seamless scaling, the abstractions that allow Kubernetes communication between applications, further elaboration on CNI drivers, load balancing, DNS, and how to expose applications to the outside world.

[Read the ebook.](https://info.rancher.com/kubernetes-networking-deep-dive)

### Load Balancing with Kubernetes – Kubernetes Master Class

Author of our ebook [“Diving Deep into Kubernetes Networking”](https://info.rancher.com/kubernetes-networking-deep-dive), Rancher Principal Software Engineer Murali Paluru presents this 2-hour video on key networking topics, including:

- Difference between Kubernetes Load Balancer Service and Ingress
- Kubernetes ingress capabilities
- An overview of various deployment models for ingress controllers
- Best practices for Load Balancer integration with external DNS
- How Rancher makes Kubernetes Ingress and Load Balancer configuration experience easier for an end-user