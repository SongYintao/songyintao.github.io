---
title: The layer of the OSI Model Illustrated
subtitle: OSL Model review
layout: post
tag: [network,OSI]
---

[参考连接](https://www.lifewire.com/layers-of-the-osi-model-illustrated-818017)

### Open Systems Interconnection (OSI) Model

The [Open Systems Interconnection (OSI) model](https://www.lifewire.com/layers-of-the-osi-model-illustrated-818017) defines a networking framework to implement protocols in layers, with control passed from one layer to the next. It is primarily used today as a teaching tool. It conceptually divides [computer network](https://www.lifewire.com/what-is-computer-networking-816249) architecture into 7 layers in a logical progression. The lower layers deal with electrical signals, chunks of [binary data](https://www.lifewire.com/working-with-binary-and-hexadecimal-numbers-816247), and routing of these data across networks. Higher levels cover network requests and responses, representation of data, and network protocols as seen from a user's point of view. 

The OSI model was originally conceived as a standard architecture for building network systems and indeed, many popular network technologies today reflect the layered design of OSI.



#### 1.Physical Layer

At Layer 1, the Physical layer of the OSI model is responsible for ultimate transmission of digital data [bits](https://www.lifewire.com/definition-of-bit-816250) from the Physical layer of the sending (source) device over network communications media to the Physical layer of the receiving (destination) device. Examples of Layer 1 technologies include [Ethernet cables](https://www.lifewire.com/what-is-an-ethernet-cable-817548) and [Token Ring networks](https://www.lifewire.com/what-is-token-ring-817952). Additionally, [hubs](https://www.lifewire.com/ethernet-and-network-hubs-816358) and other [repeaters](https://www.lifewire.com/definition-of-repeater-816359) are standard network devices that function at the Physical layer, as are cable connectors.

At the Physical layer, data are transmitted using the type of signaling supported by the physical medium: electric voltages, radio frequencies, or pulses of infrared or ordinary light.



#### 2.Data Link Layer

When obtaining data from the Physical layer, the Data Link layer checks for physical transmission errors and packages bits into data "frames". The Data Link layer also manages physical addressing schemes such as [MAC](https://www.lifewire.com/media-access-control-mac-817973)addresses for Ethernet networks, controlling access of any various network devices to the physical medium. Because the Data Link layer is the single most complex layer in the OSI model, it is often divided into two parts, the "Media Access Control" sublayer and the "Logical Link Control" sublayer.



#### 3.Network Layer

The Network layer adds the concept of routing above the Data Link layer. When data arrives at the Network layer, the source and destination addresses contained inside each frame are examined to determine if the data has reached its final destination. If the data has reached the final destination, this Layer 3 formats the data into packets delivered up to the Transport layer. Otherwise, the Network layer updates the destination address and pushes the frame back down to the lower layers.

To support routing, the Network layer maintains logical addresses such as [IP addresses](https://www.lifewire.com/what-is-an-ip-address-2625920) for devices on the network. The Network layer also manages the mapping between these logical addresses and physical addresses. In IP networking, this mapping is accomplished through the [Address Resolution Protocol (ARP)](https://www.lifewire.com/address-resolution-protocol-817941).



#### 4.Transport Layer

The Transport Layer delivers data across network connections. [TCP](https://www.lifewire.com/transmission-control-protocol-and-internet-protocol-816255) is the most common example of a Transport Layer 4 [network protocol.](https://www.lifewire.com/definition-of-protocol-network-817949) Different transport protocols may support a range of optional capabilities including error recovery, flow control, and support for re-transmission.

#### 5. Session Layer

The Session Layer manages the sequence and flow of events that initiate and tear down network connections. At Layer 5, it is built to support multiple types of connections that can be created dynamically and run over individual networks.



#### 6. Presentation Layer

The Presentation layer is the simplest in function of any piece of the OSI model. At Layer 6, it handles syntax processing of message data such as format conversions and encryption / decryption needed to support the Application layer above it.



#### 7. Application Layer

The Application layer supplies network services to end-user applications. Network services are typically protocols that work with user's data. For example, in a Web browser application, the Application layer protocol [HTTP](https://www.lifewire.com/hypertext-transfer-protocol-817944) packages the data needed to send and receive Web page content. This Layer 7 provides data to (and obtains data from) the Presentation layer.