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

These specifications assume that a RADIUS client can directly contact a RADIUS server, which is the normal "forward" path for packets between a client and server.  However, the reverse connection is not always possible.  If a RADIUS server wishes to act as a CoA client, and send CoA packets to the NAS (CoA server), the "reverse" path is often blocked by a firewall, NAT gateway, etc.

When the reverse path is blocked, it means that it is impossible to use CoA packets in a roaming / federated environment such as Eduroam or OpenRoaming.  There are resulting business and security issues, such as the inability to disconnect an online user when their account has been terminated.

This specification solves that problem.  The solution is to simply allow CoA packets to go in "reverse" down an existing RADIUS/TLS connection.  That is, when a NAS connects to a RADIUS server it normally sends request packets (Access-Request, etc.) and expects to receive response packets (Access-Accept, etc.).  This specification extends RADIUS by permitting a RADIUS server to re-use an existing TLS connection to send CoA packets to the NAS, and permitting the NAS to send CoA response packets to the RADIUS server over that same connection.

There are additional considerations for proxies.  While [RFC8559] describes CoA proxying, there are still issues which need to be addressed for the "reverse CoA" use-case.  This specification describes how a proxy can implement "reverse CoA" proxying, including signalling necessary to negotiate this functionality.

# Terminology

{::boilerplate bcp14}

* CoA

> Change of Authorition packets.  For brevity, when this document refers to "CoA" packets, it means either or both of CoA-Request and Disconnect-Request packets.

* RADIUS

> The Remote Authentication Dial-In User Service protocol, as defined in [RFC2865], [RFC2865], and [RFC5176] among others.

* RADIUS/UDP

> RADIUS over the User Datagram Protocol as define above.

* RADIUS/TCP

> RADIUS over the Transmission Control Protocol [RFC6613]

* RADIUS/TLS

> RADIUS over the Transport Layer Security protocol [RFC6614]

* RADIUS/DTLS

> RADIUS over the Datagram Transport Layer Security protocol  [RFC7360]

* SRADIUS

> The Secure RADIUS protocol, as defined in this document.  We use SRADIUS interchangeable for TLS and for DTLS transport.

* TLS

> the Transport Layer Security protocol.  Generally when we refer to TLS in this document, we are referring to RADIUS/TLS and/or RADIUS/DTLS.

# Reverse CoA between Client and Server

The reverse CoA functionality must be enabled somehow, either administratively via a configuration flag, or dynamically via per-connection negotiation.

## Configuration Flag

Clients and servers SHOULD  be administratively configured with a flag which indicates that the other party supports the reverse CoA functionality.  Such a flag can be used where the parties are known to each other.

This flag is best used for a direct connection between a NAS and a RADIUS server.  In that situation, (subject to [RFC5176] requirements), the NAS will be able to accept and process all CoA packets, no matter what their contents.  That is, the NAS will not act as a proxy to forward packets.

However, a NAS may contain multiple and independent processing modules.  In that situation, the NAS may have multiple RADIUS/TLS connections from the same source IP address.  It may not be always possible to send CoA packets down any RADIUS/TLS connection with that source IP, and have them go to the correct sub-module of the NAS.

Instead, the RADIUS server must somehow determine that a particular connection is for a particular subsystem, and only send CoA packets when the subsystem matches.

i.e. Accounting-Request contains Operator-NAS-Identifier ([RFC8559] Section 3.4).  And each connection sends a Status-Server packet containing the same Operator-NAS-Identifier.

## Dynamic Negotiation

[RFC6614] permits the use of Status-Server over RADIUS/TLS connections, as an application-level watchdog timer [RFC3539].  We leverage that functionality here.

* Status-Server SHOULD contain one or more Operator-Name attributes [RFC5580] Section 4.1.
* Operator-Name is '+' and then 'realm' to add a realm
  * bare `+` is "accepts all realms"
* Operator-Name is '-' and then 'realm' to delete a specific realm
  * bare `-` is "delete all realms"
  * these are processed in order, so "-" followed by "+example.org" means "delete all realms, and add only one for example.org"

AND/OR

* Operator-NAS-Identifier
  * if this exists and contains a value, it will accept all user sessions for that NAS.
  * more than one can exist in the same packet :(


# Proxying Reverse CoA

Proxying is still done as per [RFC8559] Section 3.3.

* look up the realm in the reverse CoA realm table to find an outgoing connection (allowing wildcard)
  * send the packet on the found connection
  * Where the realm is unknown or not routable, the proxy MUST return a NAK packet that
   contains an Error-Cause Attribute having value 502 ("Request Not
   Routable").

We also update [RFC8559] Section 3.4 to permit Status-Server packets to contain zero or one instance of the Operator-NAS-Identifier attribute.

We extend [RFC8559] Section 3.3 to allow also allow proxying via Operator-NAS-Identifier.

* if there's an Operator-NAS-Identifier in the packet, look that up in the NAS routing table
  which is another "key to connection" mapping table as with Operator-Name


# Implementation Status

FreeRADIUS supports CoA proxying using vendor-specific attributes.  It also permits RADIUS clients to send Status-Server packets over a RADIUS/TLS connection which contain Operator-Name.  This information is used to determne which realms are accessible via reverse CoA over which RADIUS/TLS connection.

Cisco supports it as of Cisco IOS XE Bengaluru 17.6.1.  https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sec/b_176_sec_9300_cg/configuring_radsec.pdf

Aruba documentation states that "Instant supports dynamic CoA (RFC 3576) over RadSec and the RADIUS server uses an existing TLS connection opened by the Instant AP to send the request." https://www.arubanetworks.com/techdocs/Instant_83_WebHelp/Content/Instant_UG/Authentication/ConfiguringRadSec.htm

# Privacy Considerations

This document does not change or add any privacy considerations over previous RADIUS specifications.

# Security Considerations

This document increases network security by removing the requirement for non-standard "reverse" paths for CoA-Request and Disconnect-Request packets.

# IANA Considerations

TBD - new RADIUS attribute

Define a new registry for Operator-Name.  3 columns:

* 1 character Namespace ID
* description
* reference

```
0,TADIG,RFC5580
1,Realm,RFC5580
2,E212,RFC5580
3,ICC,RFC5580
+,Realm Add,(this document)
-,Realm Delete,(this document)
```

# Acknowledgements

Thanks to Heikki Vatiainen for doing a preliminary implementation in Radiator, and for verifying interoperability with NAS equipment.

# Changelog


--- back

