---
title: IPvD Protocol V1 Specs
---

# IPvD (Internet Protocol Version Decentralised)

All packets must be in the order they appear in on the list.

## Why?

### Security

Because IPvD uses content based addressing, end to end encryption can be used as soon as the connection has started. This almost eliminates man-in-the-middle attacks.
Also, IDRP (see below) has been designed to verify that the route can be accessed by the node preventing route hijacking.

### Backwards Compatibility

As each address has a version specifier at the start of the address, IPvD can handle breaking changes as the server and client should be able to detect the highest common protocol version and use that.

### Flexibility

The IPvD address has no fixed size, so when SHA-512 becomes obsolete or when quantum security comes around servers can upgrade without downtime. As any client can connect with every protocol version. Also, because every IP can be generated offline, remote locations (like mars) can set up infrastructure without waiting 20 minutes to get an IP.

### Decentralisation

Because there is no central authority anyone can get online (as long as they have someone willing to link their systems)

## Addresses

### To obtain an address

- Generate a ECC-384 Keypair
- Save the public key.
- Hash your public key with the SHA-512 algorithm
- Encode the hash in extended Base32
- Add Version Specifier

### Address Format

- Address Must contain "v1-" at start.
- Last Section of address must be the SHA-512 hash of your Public Key represented in base32 format.

Example:

```
v1-266SL62DWSZS5TFOIBMLFJXAOMSBUA4WHZQPRB7BINLWRLEIUS3BTX7RDVD2PEKJY6KSCUT3A4PRS5H7O553JBPB6HDOA3TASUJ63BY=
```

## Packet Structure

All IPvD packets must contain (In this order):

- Sender Address Size (uint16) (Big Byte order)
- Sender Address (utf-8)
- Recipient Address Size (uint16) (Big Byte order)
- Recipient Address (utf-8)
- Signature Size (uint16) (Big Byte order)
- Signature (bytes)
- Hops (uint8) (Big Byte order)
- Type (uint8) (Big Byte order)
- Data (bytes)

The data must be encrypted with the recipients public key. <br>
The Signature must be the encrypted data signed by the senders private key.
All IPvD protocols must have a type.
Current Types:

| Protocol | Type |
|--------|----|
| IDRP | 1 |
| IDHP | 2 |
| TCP | 3 |
| UDP | 4 |
| PING | 5 |

### All IPvD packets must have the EtherType 0xDA11

<br>

## Types of Device

| Name Used | Description |
|-|-|
| End Device or Device | Generic device should be assumed unless otherwise stated |
| NAT-Device | A Device that separates a the internet and a private network |
| Semi-Node | An IDRP router that only runs in semi-mode |
| Full-Node | An IDRP router that only runs in full-mode |

<br>

## Network Types

| Network Type | Description |
|-----------|-------------|
|Public  | A network that every address can be accessable by anyone on the internet|
|Private | A network where only the address can be accessed by the other devices on the network*

*Unless otherwise stated in the NAT-device

<br>

## Routing and Discovery

### PDP (Peer Discovery Protocol)

PDP is a Level 2 Protocol

It has an ethertype of 0xda12

When a client connects to a new network (and every minute after), It must broadcast a PDP packet (details Below) and await a response. Then, the info must be stored for the interface the packet was recieved on.<br>

#### Request Packet

- Sender Size (uint16)
- Sender Address (utf-8)

#### Response Packet

- Return Address Size (uint16) (Big Byte order)
- Return Address (utf-8)
- Managed (bool)  [If this flag is set the following must also be added]
  - Default Route MAC Address (utf-8) (12 Chars)
  - Main DNS Server Address Size (uint16) (Big Byte order)
  - Main DNS Server Address (utf-8)
  - Alt DNS Server Address Size (uint16) (Big Byte order)
  - Alt DNS Server Address (utf-8)
- Type (uint8)
  - Must be 1 if the connection is to a end-device
  - Must be 2 if the connection is to a NAT device
  - Must be 3 if the connection is to a Semi-Node
  - Must be 4 if the connection is to a Full-Node

Any device that forwards traffic should have a default interface to send all unkown traffic.

External facing interfaces should never allow a client that has the managed flag set.

Switches must forward PDP packets to the known NAT-Device. If a switch is used externally it must never forward PDP packets but respond as a node according to the node status.

### ARP (Address Resolution Protocol)

ARP is a level 2 Protocol

It has an ethertype of 0xda13

ARP should only be used in internal networks. When a client on a internal network does not know an address it must broadcast a ARP packet. This are packet must contain the following:

- IPvD Address Size (uint16) (Big Byte order)
- IPvD Address (utf-8)

The responder must respond with:

- MAC Address (utf-8) (12 Chars)

If the NAT-device detects a ARP packet, and the requested addresses are on the network the NAT-device connects to. It should not respond. If the requested address is not on the network the NAT-device must respond its MAC address. Then, It must relay any packets (that a not for the NAT-Device) to the default route or any applicable routes.

### IDRP (IpvD Routing Protocol)

IDRP is an IPvD Protocol.

IDRP is the alternative to BGP, There are two types of nodes:

| Node Type | Description |
|-----------|-------------|
|Full Node  | This node must save all routes (save the address and next hop.)|
|Semi Node  | This node should only cache routes for direct peers.|

Node behaivor:
- If a node recieves a packet for an unknown address that node should forward that packet to a trusted node (user defined default route)
- If any node receives a route (except from a full node to a semi-node) that route must be relayed to all other full nodes
- Semi-nodes must discard all non-direct routes (if the relayed flag is set)
- If receiving a route from a node, the recieving node should verify the connection by sending an IDRP ping packet
- All IPvD packets must contain a “hops” section (the default should be 128). Every node the packet passes through must decrease the value by 1
- If a node receives a packet with a hop number of 0, the packet must be discarded. 
- All routes must have a TTL (Time to live). This value is in seconds, If the difference between the time the route was saved and the current time is longer than the time to live. The route should be deleted. A TTL must never exceed 2^31-1.
- Every time a node receives a route it must forward it on to all other nodes. Full nodes must never send routes to semi-nodes. Semi-nodes must send send all routes received to the default node. But semi-nodes should not cache this info.

IDRP Packet IPvD:
- IPvD Route Address Size (uint16) (Big Byte order)
- IPvD Route Address (utf-8)
- MAC Address Hop (utf-8) (12 Chars)
- TTL (uint32) (Big Byte order)

IDRP is an IPvD Protocol.

### Ping

Send a ping packet and await a response.
The random data allows signing of the packet insuring authenticity.

Ping Packet:
- random 16 byte string (utf-8)

Response:
- random 16 byte string (utf-8)

<br>

## Connection Initiation

### IDHP (IpvD Handshake Protocol)

Before any connection IDHP must be utilised.
IDHP is the only IPvD protocol where messages cannot be encrypted as IDHP is where the key exchange takes place.

The client must send a IDHP packet containing the clients public key and the server must respond the same. Both of the devices must verify the address and the public key by hashing the public key and verify that last section of the address. The IDHP response is the same as the orignally sent packet but with the responders public key.

IDHP Packet:
- Public key Size (uint16) (Big Byte order)
- Public key (utf-8

The client and server (when connecting) must determine the highest common version for the protocol to use.

<br>

### TCP and UDP

TCP and UDP should not deviate from IPv6.

## Licence and modification

```
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.
```

```
This is free and unencumbered software released into the public domain.

Anyone is free to copy, modify, publish, use, compile, sell, or
distribute this software, either in source code form or as a compiled
binary, for any purpose, commercial or non-commercial, and by any
means.

In jurisdictions that recognize copyright laws, the author or authors
of this software dedicate any and all copyright interest in the
software to the public domain. We make this dedication for the benefit
of the public at large and to the detriment of our heirs and
successors. We intend this dedication to be an overt act of
relinquishment in perpetuity of all present and future rights to this
software under copyright law.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR
OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
OTHER DEALINGS IN THE SOFTWARE.

For more information, please refer to <http://unlicense.org/>
```

<a href="https://github.com/JKincorperated/JKincorperated.github.io">Visit the github Repo</a>
