---
title: Reverse CoA in RADIUS
abbrev: Reverse CoA
docname: draft-dekok-radext-reverse-coa-00

stand_alone: true
ipr: trust200902
area: Internet
wg: RADEXT Working Group
kw: Internet-Draft
cat: std
submissionType: IETF

pi:    # can use array (if all yes) or hash here
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:

- ins: A. DeKok
  name: Alan DeKok
  org: FreeRADIUS
  email: aland@freeradius.org

- ins: V. Cargatser
  name: Vadim Cargatser
  org: Cisco
  email: vcargats@cisco.com

normative:
  BCP14: RFC8174
  RFC2865:
  RFC3539:
  RFC5580:
  RFC8559:

informative:
  RFC5176:
  RFC6613:
  RFC6614:
  RFC7360:

venue:
  group: RADEXT
  mail: radext@ietf.org
  github: freeradius/reverse-coa.git

--- abstract

This document defines a "reverse change of authorization (CoA)" path for RADIUS packets.  This specification allows a home server to send CoA packets in "reverse" down a RADIUS/TLS connection.  Without this capability, it is impossible for a home server to send CoA packets to a NAS which is behind a firewall or NAT gateway,

--- middle

# Introduction

[RFC5176] defines the ability to change a users authorization, or disconnect the user via what are generally called "Change of Authorization" or "CoA" packets.  This term refers to either of the RADIUS packet types CoA-Request or Disconnect-Request.  The initial transport protocol for all RADIUS was the User Datagram Protocol (UDP).

[RFC6614] updated previous specifications to allow packets to be sent over the Transport Layer Security (TLS) protocol.  Section 2.5 of that document explicitly allows all packets (including CoA) to be sent over a TLS connection:

```
Due to the use of one single TCP port for all packet types, it is
required that a RADIUS/TLS server signal which types of packets are
supported on a server to a connecting peer.  See also Section 3.4 for
a discussion of signaling.
```

These specifications assume that a RADIUS client can directly contact a RADIUS server, which is the normal "forward" path for packets between a client and server.  However, the reverse connection is not always possible or even desirable.  If a RADIUS server wishes to act as a CoA client, and send CoA packets to the NAS (CoA server), the "reverse" path is often blocked by a firewall, NAT gateway, etc.  That is, a RADIUS server has to be reachable by a NAS, but there is usually a requirement that the NAS is not reachable from any public system.

As the reverse path is usally blocked, it means that it is in general possible only to send CoA packets to a NAS when the NAS and RADIUS server share the same private network (private IP space or IPSec).  Even though [RFC8559] defines CoA proxying, it does not solve the issue of NAS reachability.

The problem is most evident in a roaming / federated environment such as Eduroam or OpenRoaming.  It is in general impossible for a home server to signal the NAS to disconnect a user.  There is no direct reverse path from the home server to the NAS, as the NAS is not publicly addressible.  Even if there was a public reverse path, it would generally be unknowable, as intermediate proxies can (and do) attribute rewriting to hide NAS identies.

These limitations result in business and security issues, such as the inability to disconnect an online user when their account has been terminated.

This specification solves that problem.  The solution is to simply allow CoA packets to go in "reverse" down an existing RADIUS/TLS connection.  That is, when a NAS connects to a RADIUS server it normally sends request packets (Access-Request, etc.) and expects to receive response packets (Access-Accept, etc.).  This specification extends RADIUS/TLS by permitting a RADIUS server to re-use an existing TLS connection to send CoA packets to the NAS, and permitting the NAS to send CoA response packets to the RADIUS server over that same connection.

We note that while this document specifically mentions RADIUS/TLS, it should be possible to use the same mechanisms on RADIUS/DTLS [RFC7360].  However at the time of writing this specification, no implementations exist for "reverse CoA" over RADIUS/DTLS.  We also note that while (in theory) this same mechanism could be used for RADIUS/UDP and RADIUS/TCP, there is no value in defining "reverse CoA" for those transports.  Therefore for practial purposes, "reverse CoA" means RADIUS/TLS.

There are additional considerations for proxies.  While [RFC8559] describes CoA proxying, there are still issues which need to be addressed for the "reverse CoA" use-case.  This specification describes how a proxy can implement "reverse CoA" proxying, including signalling necessary to negotiate this functionality.

# Terminology

{::boilerplate bcp14}

* CoA

> Change of Authorization packets.  For brevity, when this document refers to "CoA" packets, it means either or both of CoA-Request and Disconnect-Request packets.

* RADIUS/TLS

> RADIUS over the Transport Layer Security protocol [RFC6614]

* RADIUS/DTLS

> RADIUS over the Datagram Transport Layer Security protocol  [RFC7360]

# Concepts

There are a number of related problems which need to be solved in order for reverse CoA to work:

* routing across a proxy network to a particular visited server.  The proxies are often managed by unrelated administrators.

* routing from the visited server to a particular NAS.  There is usually no proxying here, or if there is, it's all under one administration

* signalling from client to server that this functionality is available on a connection.

We discuss these issues in reverse order.

# Capability configuration and negotiation

There are two ways to enable this functionality.  The first is a simple static configuration between client and server, where both are configured to allow reverse CoA.  The second method is via per-connection signalling between client and server.

## Configuration Flag

Clients and servers SHOULD have a configuration a flag which indicates that the other party supports the reverse CoA functionality.  That is, the client has a per-server flag enabling (or not) reverse CoA functionality.  The server has a similar per-client flag.

The flag can be used where the parties are known to each other, i.e. not for dynamic discovery as with [RFC7585].  The flag can also be used where there is only one subsystem on the client which sends packets.

For example, a NAS may have one public IP address, but internally it may contain multiple and independent processing modules.  In that situation, the NAS may have multiple RADIUS/TLS connections from the same source IP address.  It may not be always possible to send CoA packets down any RADIUS/TLS connection with that source IP, and have them received by the correct sub-module of the NAS.

For these reasons, it is useful to allow for dynamic negotiation of reverse coa on a per-connection basis.

## Dynamic Negotiation

The reverse CoA functionality can be signalled on a per-connection basis by the client sending a Status-Server packet when it first opens a connection to a server.  This packet contains a Capability attribute, with value "Reverse-CoA".  The existence of this attribute in a Status-Server packet indicates that the client supports reverse CoA over this connection.  The Status-Server packet is the first packet sent when the connection is opened, in order to perform per-connection signalling.  A server which does not implement reverse CoA simply ignores this attribute, as per [RFC2865] Section 5.

A server implementing reverse CoA does not need to signal the NAS in response, to indicate that it is also has this capability.  If the server never sends reverse CoA packets, then such signalling is unnecessary.  If the server does send reverse CoA packets, then the packets themselves serve as sufficiant signalling.

The NAS may send additional Status-Server packets down the same connection, as per [RFC3539].  These packets do not need to contain the Capability attribute, so it can generally be omitted.  That is, there is no need to signal the addition or removal of reverse CoA functionality during the lifetime of one connection.

This Status-Server packet may also contain multiple Operator-Name attributes.  The use of this attribute is discussed in more detail below.

# Routing of Reverse CoA packets

There are multiple types of reverse CoA routing which are possible:

* direct connection NAS to server, NAS has no subsystems, and there is no realm routing,
* direct connection NAS to server, but the NAS has multiple subsystems, and there is no realm routing,
* Proxy to proxy connections, all CoA routing is done on Operator-Name, as per [RFC8559] Section 5.2.

In the first scenario, CoA packets are routed as described in [RFC5176], except that the packets are no permitted to go in "reverse" over a RADIUS/TLS connection.

In the seecond scenario, the Operator-NAS-Identifier attribute is used by the visited server to distinguish connections, and to route reverse CoA packets to the correct connection, and thus the correct NAS subsystem.

In the final scenario, we have server to server proxying.  Routing is done using the Operator-Name attribute, as defined in [RFC8559].

We discuss these scenarios in more detail below.  We also address the issue of identifying a particular NAS or NAS subsystem, in order for a reverse CoA proxy to make the association between a connection with a particular NAS.

## Direct NAS to server connections

In the simplest case, the server simply routes reverse CoA packets over any available connection (NAS to server).  All connections can be treated as identical, in that they connect to the same logical endpoint.  Any CoA packet can be sent down any connection, and there is no further routing of packets past the NAS.

Routing is done in "reverse" over a RADIUS/TLS connection.  All routing decisions are made as defined in [RFC5176].  The server may have connections from multiple NASses, and packets are routed to the correct NAS.  However, multiple connections from one NAS are treated as identical.  Any connection can be used to send reverse CoA packets to that NAS.

For any more complex scenario, we have to first determine a way to identify the NAS.

## NAS Identification

RADIUS/TLS as defined in [RFC6614] does not need to identify NASes, other than as known / unknown, via certificates or PSKs.  It doesn't matter which NAS a connection belongs to, as the NAS is originating all traffic.  The server just takes the packets, and handles them.  i.e. if two connections have packets with the same (or different) NAS-Identifier, it doesn't matter.  The server just stores that data, and does not examine it any further.

CoA proxying as defined in [RFC8559] routes packets globally based on Operator-Realm, or locally based on Operator-NAS-Identifier.  In which case the local server has a well-defined mapping between CoA server (NAS) and Operator-NAS-Identifier.  This mapping is local to the server, and is not known by the NAS.  The mapping is likely created and managed by the RADIUS server which is closest to the NAS.  This association could be static, and based on NAS IP address, or it could be based on any other information which is available to the administrator.

For this specification, we have the problem where multiple NASes may be connecting to a server behind a NAT gateway.  At which point we need to route packets to the correct NAS, which means that we need to identify the NAS.  In RADIUS, the only identity a NAS has is its source IP.  For RADIUS/TLS, the NAS is _permitted_ based on its TLS parameters (cert, etc.).  But it has no _identity_ as such.

For reverse CoA to work, the server sending the CoA packets to the NAS has to identify that NAS, on a per-connection basis.  The solution here is to note that [RFC5997] Section 5 says:

```
A Status-Server packet SHOULD contain one of (NAS-IP-Address or NAS-IPv6-Address), or NAS-Identifier, or both NAS-Identifier and one of (NAS-IP-Address or NAS-IPv6-Address).
```

We suggest here that a server implementing reverse CoA simply make an association between the NAS identification attributes available in a Status-Server packet, and the RADIUS/TLS connection on which that packet was received.  There may be multiple RADIUS/TLS connections from the NAS, so this association is not one-to-one.  An implementation MUST support multiple connections the same NAS.

When the server needs to send a reverse CoA packet to the NAS, it can look at the CoA packet for the same NAS identification attributes, selects one of the available connections, and sends the packet down the chosen connection.

If no connections to the NAS are available, the server MUST return a NAK packet that contains an Error-Cause Attribute having value 502 ("Request Not Routable"), as per [RFC9559] Section 4.3.1.

When multiple connections to the same NAS are available, the choice of which connection to use is implementation-specific.  For the purpose of this specification, all of the connections are identical in functionality.  It is RECOMMENDED that implementations use fail-over and load-balancing techniques similar to those used for proxying normal RADIUS packets.

## NAS with Multiple Subsystems

Once we have a way to identify a NAS, it is then trivial to differentiate multiple subsystems within one NAS.  We simply require that each subsystem use a unique value for the NAS-Identifier attribute.

The subsystems may use different values for the NAS-IP-Address or NAS-IPv6-Address attributes, but at that point they are essentially different NASes.

If the NAS can route reverse CoA packets internally between different subsystems, it SHOULD use the same identification attributes in Status-Server packets for outgoing RADIUS/TLS connections.

If the NAS cannot route reverse CoA packets internally between different subsystems, it MUST use different values for identification attributes in Status-Server packets for outgoing RADIUS/TLS connections.

## Proxying Reverse CoA Packets

Proxying is still done as per [RFC8559] Section 3.3.

* look up the realm in the reverse CoA realm table to find an outgoing connection (allowing wildcard)
  * send the packet on the found connection
  * Where the realm is unknown or not routable, the proxy MUST return a NAK packet that contains an Error-Cause Attribute having value 502 ("Request Not Routable").

[RFC6614] permits the use of Status-Server over RADIUS/TLS connections, as an application-level watchdog timer [RFC3539].  We extend the functionality to allow signalling about realm availability.

* Status-Server SHOULD contain one or more Operator-Name attributes [RFC5580] Section 4.1.
* Operator-Name is '+' and then 'realm' to add a realm
  * bare `+` is "accepts all realms"
* Operator-Name is '-' and then 'realm' to delete a specific realm
  * bare `-` is "delete all realms"
  * these are processed in order, so "-" followed by "+example.org" means "delete all realms, and add only one for example.org"

# Implementation Status

FreeRADIUS supports CoA proxying using vendor-specific attributes.  It also permits RADIUS clients to send Status-Server packets over a RADIUS/TLS connection which contain Operator-Name.  This information is used to determne which realms are accessible via reverse CoA over which RADIUS/TLS connection.

Cisco supports it as of Cisco IOS XE Bengaluru 17.6.1.  https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sec/b_176_sec_9300_cg/configuring_radsec.pdf

Aruba documentation states that "Instant supports dynamic CoA (RFC 3576) over RadSec and the RADIUS server uses an existing TLS connection opened by the Instant AP to send the request." https://www.arubanetworks.com/techdocs/Instant_83_WebHelp/Content/Instant_UG/Authentication/ConfiguringRadSec.htm

# Privacy Considerations

This document does not change or add any privacy considerations over previous RADIUS specifications.

# Security Considerations

This document increases network security by removing the requirement for non-standard "reverse" paths for CoA-Request and Disconnect-Request packets.

# IANA Considerations

TBD - new RADIUS attribute - Capability

User Operator Namespace Identifier namespace.

```
+,Realm Add,(this document)
-,Realm Delete,(this document)
```

# Acknowledgements

Thanks to Heikki Vatiainen for testing a preliminary implementation in Radiator, and for verifying interoperability with NAS equipment.

# Changelog


--- back

