---
title: Reverse CoA in RADIUS
abbrev: Reverse CoA
docname: draft-ietf-radext-reverse-coa-02

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
  RFC7585:
  RFC8559:

informative:
  RFC5176:
  RFC6614:
  RFC7360:

venue:
  group: RADEXT
  mail: radext@ietf.org
  github: freeradius/reverse-coa.git

--- abstract

This document defines a "reverse change of authorization (CoA)" path for RADIUS packets.  This specification allows a home server to send CoA packets in "reverse" down a RADIUS/TLS connection.  Without this capability, it is impossible for a home server to send CoA packets to a NAS which is behind a firewall or NAT gateway.  The reverse CoA functionality extends the available transport methods for CoA packets, but it does not change anything else about how CoA packets are handled.

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

These specifications assume that a RADIUS client can directly contact a RADIUS server, which is the normal "forward" path for packets between a client and server.  However, it is not always possible for the RADIUS server to send CoA packets to the RADIUS client. If a RADIUS server wishes to act as a CoA client, and send CoA packets to the NAS (CoA server), the "reverse" path can be blocked by a firewall, NAT gateway, etc.  That is, a RADIUS server has to be reachable by a NAS, but there is usually no requirement that the NAS is reachable from a public system.  To the contrary, there is usually a requirement that the NAS is not publicly accessible.

This scenario is most evident in a roaming / federated environment such as Eduroam or OpenRoaming.  It is in general impossible for a home server to signal the NAS to disconnect a user.  There is no direct reverse path from the home server to the NAS, as the NAS is not publicly addressible.  Even if there was a public reverse path, it would generally be unknowable, as intermediate proxies can (and do) attribute rewriting to hide NAS identies.

These limitations can result in business losses and security problems, such as the inability to disconnect an online user when their account has been terminated.

As the reverse path is usally blocked, it means that it is in general possible only to send CoA packets to a NAS when the NAS and RADIUS server share the same private network (private IP space or IPSec).  Even though [RFC8559] defines CoA proxying, that specification does not address the issue of NAS reachability.

This specification solves that problem.  The solution is to simply allow CoA packets to go in "reverse" down an existing RADIUS/TLS connection.  That is, when a NAS connects to a RADIUS server it normally sends request packets (Access-Request, etc.) and expects to receive response packets (Access-Accept, etc.).  This specification extends RADIUS/TLS by permitting a RADIUS server to re-use an existing TLS connection to send CoA packets to the NAS, and permitting the NAS to send CoA response packets to the RADIUS server over that same connection.

We note that while this document specifically mentions RADIUS/TLS, it should be possible to use the same mechanisms on RADIUS/DTLS [RFC7360].  However at the time of writing this specification, no implementations exist for "reverse CoA" over RADIUS/DTLS.  As such, when we refer to "TLS" here, or "RADIUS/TLS", we implicitly include RADIUS/DTLS in that description.

We also note that while this same mechanism could theoretically be used for RADIUS/UDP and RADIUS/TCP, there is no value in defining "reverse CoA" for those transports.  Therefore for practial purposes, "reverse CoA" means RADIUS/TLS and RADIUS/DTLS.

There are additional considerations for proxies.  While [RFC8559] describes CoA proxying, there are still issues which need to be addressed for the "reverse CoA" use-case.  This specification describes how those systems can implement "reverse CoA" proxying, including processing packets through both an intermediate proxy network, and at the visited network.

# Terminology

{::boilerplate bcp14}

* CoA

> Change of Authorization packets.  For brevity, when this document refers to "CoA" packets, it means either or both of CoA-Request and Disconnect-Request packets.

* ACK

> Change of Authorization "positive acknowlegement" packets.  For brevity, when this document refers to "ACK" packets, it means either or both of CoA-ACK and Disconnect-ACK packets.

* NAK

> Change of Authorization "negative acknowlegement" packets.  For brevity, when this document refers to "NAK" packets, it means either or both of CoA-NAK and Disconnect-NAK packets.

* RADIUS/TLS

> RADIUS over the Transport Layer Security protocol [RFC6614]

* RADIUS/DTLS

> RADIUS over the Datagram Transport Layer Security protocol  [RFC7360]

* TLS

> Either RADIUS/TLS or RADIUS/DTLS.

* reverse CoA

> CoA, ACK, or NAK packets sent over a RADIUS/TLS or RADIUS/DTLS connection which was made from a RADIUS client to a RADIUS server.

# Concepts

The reverse CoA functionality is based on two additions to RADIUS.  The first addition is a configuration and signalling, to indicate that a RADIUS client is capable of accepting reverse CoA packets.  The second addition is an extension to the "reverse" routing table for CoA packets which was first described in Section 2.1 of [RFC8559].

# Capability Configuration and Signalling

In order for a RADIUS server to send reverse CoA packets to a client, it must first know that the client is capable of accepting these packets.

Clients and servers implementing reverse CoA MUST have a configuration flag which indicates that the other party supports the reverse CoA functionality.  That is, the client has a per-server flag enabling (or not) reverse CoA functionality.  The server has a similar per-client flag.

The flag can be used where the parties are known to each other.  The flag can also be used in conjunction with dynamic discovery ([RFC7585]), so long as the server associates the flag with the client identity and not with any particular IP address.  That is, the flag can be associated with any method of identifying a particular client such as TLS-PSK identity, information in a client certificate, etc.

For the client, the flag controls whether or not it will accept reverse CoA packets from the server, and whether the client will do dynamic signalling of the reverse CoA functionality.

The configuration flag allows administators to statically enable this functionality, based on out-of-band discussions with other administators.  This process is best used in an environment where all RADIUS proxies are known (or required) to have a particular set of functionality, as with a roaming consortium.

This specification does not define a way for clients and servers to negotiate this functionality on a per-connection basis.  The RADIUS protocol has little, if any, provisions for capability negotiations, and this specification is not the place to add that functionality.

Without notification, however, it is possible for clients and servers to have mismatched configurations.  Where a client is configured to accept reverse CoA packets and the next hop server is not configured to send them, no packets will be sent.  Where a client is configured to not accept reverse CoA packets and the next hop server is configured to send them, the client will silently discard these packets as per {{RFC2865, Section 3}}.  In both of those situations, reverse CoA packets will not flow, but there will be no other issues with this misconfiguration.

# Reverse Routing

The "reverse" routing table for CoA packets was first described in Section 2.1 of [RFC8559].  We extend that table here.

In our extension, the table does not map realms to home servers.  Instead, it maps keys to connections.  The keys will be defined in more detail below.  For now, we say that keys can be derived from a RADIUS client to server connection, and from the contents of a CoA packet which needs to be routed.

When the server receives a TLS connection from a client, it derives a key for that connection, and associates the connection with that key.  A server MUST support associating one particular key value with multiple connections.  A server MUST support associating multiple keys for one connection.  That is, the "key to connection" mapping is N to M.  It is not one-to-one, or 1:N, or M:1, it is many-to-many.

When the server receives a CoA packet, it derives a key from that packet, and determines if there is a connection or connections which maps to that key.  Where there is no available connection, the server MUST return a NAK packet that contains an Error-Cause Attribute having value 502 ("Request Not Routable").

As with normal proxying, a particular packet can sometimes have the choice more than one connection which can be used to reach a destination.  In that case, issues of load-balancing, fail-over, etc. are implementation-defined, and are not discussed here.  The server simply chooses one connection, and sends the reverse CoA packet down that connection.

The server then waits for a reply, doing retransmission if necessary.  For all issues other than the connection being used, reverse CoA packets are handled as defined in [RFC5176] and in [RFC8559].

That is, when the NAS and server are known to each other, [RFC5176] is followed when sending CoA packets to the NAS.  The difference is that instead of originating connections to the NAS, the server simply re-uses inbound TLS connections from the NAS.  The NAS is identified by attributes such as NAS-Identifier, NAS-IP-Address, and NAS-IPv6-Address.

When a server is proxying to another server, [RFC8559] is following when proxying CoA packets.  The "next hop" is identified either by Operator-Name for proxy-to-proxy connections.  When the CoA packet reaches a visited network, that network identifies the NAS by examining the Operator-NAS-Identifier attribute.

## Retransmits

Retransmissions of reverse CoA packets are handled identically to normal CoA packets.  That is, the reverse CoA functionality extends the available transport methods for CoA packets, it does not change anything else about how CoA packets are handled.

# Implementation Status

FreeRADIUS supports CoA proxying using Vendor-Specific attributes.

Cisco supports reverse CoA as of Cisco IOS XE Bengaluru 17.6.1 via Vendor-Specific attributes.  https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sec/b_176_sec_9300_cg/configuring_radsec.pdf

Aruba documentation states that "Instant supports dynamic CoA (RFC 3576) over RadSec and the RADIUS server uses an existing TLS connection opened by the Instant AP to send the request." https://www.arubanetworks.com/techdocs/Instant_83_WebHelp/Content/Instant_UG/Authentication/ConfiguringRadSec.htm

# Privacy Considerations

This document does not change or add any privacy considerations over previous RADIUS specifications.

# Security Considerations

This document increases network security by removing the requirement for non-standard "reverse" paths for CoA-Request and Disconnect-Request packets.

# IANA Considerations

This document requests no action from IANA.

RFC Editor: This section may be removed before publication.

# Acknowledgements

Thanks to Heikki Vatiainen for testing a preliminary implementation in Radiator, and for verifying interoperability with NAS equipment.

# Changelog

* 00 - taken from draft-dekok-radext-reverse-coa-01

* 01 - Bumped to avoid expiry

--- back

