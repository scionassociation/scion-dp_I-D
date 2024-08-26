---
title: "SCION Data Plane"
abbrev: "SCION DP"
category: info
submissiontype: independent

docname: draft-dekater-scion-dataplane-latest
v: 3
# area: "IRTF"
# workgroup: "Path Aware Networking RG"
keyword: Internet-Draft
venue:
#  group: "Path Aware Networking RG"
#  type: "Research Group"
#  mail: "panrg@irtf.org"
#  arch: "https://www.ietf.org/mail-archive/web/panrg/"
  github: "scionassociation/scion-dp_I-D"
  latest: "https://scionassociation.github.io/scion-dp_I-D/draft-dekater-scion-dataplane.html"

author:
 -   ins: C. de Kater
     name: Corine de Kater
     org: SCION Association
     email: c_de_kater@gmx.ch

 -   ins: N. Rustignoli
     name: Nicola Rustignoli
     org: SCION Association
     email: nic@scion.org

 -   ins: S. Hitz
     name: Samuel Hitz
     org: Anapaya Systems
     email: hitz@anapaya.net


normative:
  I-D.scion-cp:
    title: SCION Control Plane
    date: 2024
    target: https://datatracker.ietf.org/doc/draft-dekater-scion-controlplane/
    author:
      -
        ins: C. de Kater
        name: Corine de Kater
        org: SCION Association
      -
        ins: N. Rustignoli
        name: Nicola Rustignoli
        org: SCION Association
      -
        ins: S. Hitz
        name: Samuel Hitz
        org: Anapaya Systems
  I-D.scion-cppki:
    title: SCION Control-Plane PKI
    date: 2024
    target: https://datatracker.ietf.org/doc/draft-dekater-scion-pki/
    author:
      -
        ins: C. de Kater
        name: Corine de Kater
        org: SCION Association
      -
        ins: N. Rustignoli
        name: Nicola Rustignoli
        org: SCION Association
      -
        ins: S. Hitz
        name: Samuel Hitz
        org: Anapaya Systems
  RFC2474:
  RFC3168:
  RFC4493:
  RFC5280:
  RFC5880:
  RFC5881:
  RFC8200:

informative:
  CHUAT22:
    title: "The Complete Guide to SCION"
    date: 2022
    target: https://doi.org/10.1007/978-3-031-05288-0
    seriesinfo:
      ISBN: 978-3-031-05287-3
    author:
      -
        ins: L. Chuat
        name: Laurent Chuat
        org: ETH Zuerich
      -
        ins: M. Legner
        name: Markus Legner
        org: ETH Zuerich
      -
        ins: D. Basin
        name: David Basin
        org: ETH Zuerich
      -
        ins: D. Hausheer
        name: David Hausheer
        org: Otto von Guericke University Magdeburg
      -
        ins: S. Hitz
        name: Samuel Hitz
        org: Anapaya Systems
      -
        ins: P. Mueller
        name: Peter Mueller
        org: ETH Zuerich
      -
        ins: A. Perrig
        name: Adrian Perrig
        org: ETH Zuerich
  RFC1122:
  RFC1918:
  RFC2711:
  RFC9217:
  RFC9473:
  SCMP:
    title: SCMP Documentation
    date: 2023
    target: https://docs.scion.org/en/latest/protocols/scmp.html
    author:
      -
        ins: Anapaya
        name: Anapaya Systems
        org: Anapaya Systems
      -
        ins: ETH
        name: ETH Zuerich
        org: ETH Zuerich
      -
        ins: SCION
        name: SCION Association
        org: SCION Association
  CRYPTOBOOK:
    title: A Graduate Course in Applied Cryptography
    date: 2023
    target: https://toc.cryptobook.us/
    author:
        -
         ins: D. Boneh
         name: Dan Boneh
        -
         ins: V. Shoup
         name: Victor Shoup


--- abstract

This document describes the data plane of the path-aware, inter-domain network architecture SCION (Scalability, Control, and Isolation On Next-generation networks). One of the basic characteristics of SCION is that it gives path control to endpoints. The SCION control plane is responsible for discovering these paths and making them available as path segments to the endpoints. The role of the SCION data plane is to combine the path segments into end-to-end paths, and forward data between endpoints according to the specified path.

The SCION data plane fundamentally differs from today's IP-based data plane in that it is *path-aware*: In SCION, interdomain forwarding directives are embedded in the packet header. This document provides a detailed specification of the SCION data packet format as well as the structure of the SCION header. SCION also supports extension headers, which are additionally described. The document continues with the life cycle of a SCION packet while traversing the SCION Internet, followed by a specification of the SCION path authorization mechanisms and the packet processing at routers.

--- middle


# Introduction

SCION is a path-aware internetworking routing architecture as described in {{RFC9217}}. It allows endpoints and applications to select paths across the network to use for traffic, based on trusted path properties. SCION is an inter-domain network architecture and is therefore not concerned with intra-domain forwarding.

The data transmission order for SCION is the same as for IPv6 as defined in Introduction of {{RFC8200}}.

SCION has been developed with the following goals:

*Availability* - to provide highly available communication that can send traffic over paths with optimal or required characteristics, quickly handle inter-domain link or router failures (both on the last hop or anywhere along the path) and provide continuity in the presence of adversaries.

*Security* - to provide higher levels of trust in routing information in order to prevent traffic hijacking, reduce potential for denial-of-service and other attacks. Endpoints can decide the trust roots they wish to rely on, routing information can be unambiguously attributed to an AS, and packets are only forwarded along authorized path segments. A particular use case is to enable geofencing.

*Scalability* - to improve the scalability of the inter-domain control plane and data plane, avoiding existing limitations related to convergence and forwarding table size. The advertising of path segments is separated into a beaconing process within each Isolation Domain (ISD) and between ISDs which incurs minimal overhead and resource requirements on routers.

SCION relies on three main components:

*PKI* - To achieve scalability and trust, SCION organizes existing ASes into logical groups of independent routing planes called *Isolation Domains (ISDs)*. All ASes in an ISD agree on a set of trust roots called the *Trust Root Configuration (TRC)* which is a collection of signed root certificates in X.509 v3 format {{RFC5280}}. The ISD is governed by a set of *core ASes* which typically manage the trust roots and provide connectivity to other ISDs. This is the basis of the public key infrastructure which the SCION control plane relies upon for the authentication of messages that is used for the SCION control plane. See {{I-D.scion-cppki}}

*Control Plane* - performs inter-domain routing by discovering and securely disseminating path information between ASes. The core ASes use Path-segment Construction Beacons (PCBs) to explore intra-ISD paths, or to explore paths across different ISDs. See {{I-D.scion-cp}}

*Data Plane* - carries out secure packet forwarding between SCION-enabled ASes over paths selected by endpoints. A SCION border router reuses existing intra-domain infrastructure to communicate to other SCION routers or SCION endpoints within its AS.

This document describes the SCION Data Plane component.


## Terminology {#terms}

**Autonomous System (AS)**: An autonomous system is a network under a common administrative control. For example, the network of an Internet service provider or organization can constitute an AS.

**Core AS**: Each SCION isolation domain (ISD) is administered by a set of distinguished autonomous systems (ASes) called core ASes, which are responsible for initiating the path discovery and path construction process (in SCION called "beaconing").

**Data Plane**: The data plane (sometimes also referred to as the forwarding plane) is responsible for forwarding data packets that endpoints have injected into the network. After routing information has been disseminated by the control plane, packets are forwarded by the data plane in accordance with such information.

**Endpoint**: An endpoint is the start or the end of a SCION path. For example, an endpoint can be a host as defined in {{RFC1122}} or a [SCION IP gateway](#sig) bridging a SCION and an IP domain. This definition is based on the definition in {{RFC9473}}.

**Forwarding Key**: A forwarding key is a symmetric key that is shared between the control service (control plane) and the routers (data plane) of an AS. It is used to authenticate Hop Fields in the end-to-end SCION path. The forwarding key is an AS-local secret and is not shared with other ASes.

**Forwarding Path**: A forwarding path is a complete end-to-end path between two SCION hosts which is used to transmit packets in the data plane. It can be created with a combination of up to three path segments (an up segment, a core segment, and a down segment).

**Hop Field (HF)**: As they traverse the network, path segment construction beacons (PCBs) accumulate cryptographically protected AS-level path information in the form of Hop Fields. In the data plane, Hop Fields are used for packet forwarding: they contain the incoming and outgoing interface IDs of the ASes on the forwarding path.

**Info Field (INF)**: Each path segment construction beacon (PCB) contains a single Info field, which provides basic information about the PCB. Together with Hop Fields (HFs), these are used to create forwarding paths.

**Interface Identifier (Interface ID)**: A 16-bit identifier that designates a SCION interface at the end of a link connecting two SCION ASes, with each interface belonging to one border router. Hop fields describe the traversal of an AS by a pair of interface IDs (the ingress and egress interfaces). The Interface ID MUST be unique within each AS. Interface ID 0 is not a valid identifier as implementations MAY use it as the "unspecified" value. Interface IDs in Hop Fields are known as `ConsIngress` and `ConsEgress`, those designate the interfaces through which a packet enters (`ConsIngress`) or leaves (`ConsEgress`) the corresponding AS when that packet is traveling in the direction of the path segment's construction. The `Cons` prefix serves as a reminder that the meanings are reversed when the packet travels in the opposite direction.

**Isolation Domain (ISD)**: In SCION, Autonomous Systems (ASes) are organized into logical groups called Isolation Domains or ISDs. Each ISD consists of ASes that span an area with a uniform trust environment (e.g. a common jurisdiction). A possible model is for ISDs to be formed along national boundaries or federations of nations.

**Leaf AS**: An AS at the "edge" of an ISD, with no other downstream ASes.

**MAC**: Message Authentication Code. In the rest of this document, "MAC" always refers to "Message Authentication Code" and never to "Medium Access Control". When "Medium Access Control address" is implied, the phrase "Link Layer Address" is used.

**Path Authorization**: A requirement for the data plane is that endpoints can only use paths that were constructed and authorized by ASes in the control plane. This property is called path authorization. The goal of path authorization is to prevent endpoints from crafting Hop Fields (HFs) themselves, modifying HFs in authorized path segments, or combining HFs of different path segments.

**Path Control**: Path control is a property of a network architecture that gives endpoints the ability to select how their packets travel through the network. Path control is stronger than path transparency.

**Path Segment**: Path segments are derived from path segment construction beacons (PCBs). A path segment can be (1) an up segment (i.e. a path between a non-core AS and a core AS in the same ISD), (2) a down segment (i.e. the same as an up segment, but in the opposite direction), or (3) a core segment (i.e., a path between core ASes). Up to three path segments can be used to create a forwarding path.

**Path-Segment Construction Beacon (PCB)**: Core AS control planes generate PCBs to explore paths within their isolation domain (ISD) and among different ISDs. ASes further propagate selected PCBs to their neighboring ASes. As a PCB traverses the network, it carries path segments, which can subsequently be used for traffic forwarding.

**Path Transparency**: Path transparency is a property of a network architecture that gives endpoints full visibility over the network paths their packets are taking. Path transparency is weaker than path control.

**Peering Link**: A link between two SCION border routers of different ASes, where at least one of the two ASes is not a core AS. Two peering ASes may be in different ISDs. A peering link can be seen as a shortcut on a normal path. Peering link information is added to segment information during the beaconing process and used to shorten paths while assembling them from segments.


## Conventions and Definitions

{::boilerplate bcp14-tagged}


## Overview

The SCION data plane forwards inter-domain packets between SCION-enabled ASes. SCION routers are normally deployed at the edge of an AS, and peer with neighbor SCION routers. Inter-domain forwarding is based on end-to-end path information contained in the packet header. This path information consists of a sequence of Hop Fields (HFs). Each Hop Field corresponds to an AS on the path, and it includes an ingress interface ID as well as an egress interface ID, which uniquivocally identifies the ingress and egress interfaces within the AS. The information is authenticated with a Message Authentication Code (MAC) to prevent forgery.

This concept allows SCION routers to forward packets to a neighbor AS without inspecting the destination address and also without consulting an inter-domain forwarding table. Intra-domain forwarding and routing are based on existing mechanisms (e.g. IP). A SCION border router reuses existing intra-domain infrastructure to communicate to other SCION routers or SCION endpoints within its AS. The last SCION router at the destination AS therefore uses the destination address to forward the packet to the appropriate local endpoint.

This SCION design choice has the following advantages:

- It provides control and transparency over forwarding paths to endpoints.
- It simplifies the packet processing at routers. Instead of having to perform longest prefix matching on IP addresses which requires expensive hardware and substantial amounts of energy, a router can simply access the next hop from the packet header after having verified the authenticity of the Hop Field's MAC.


### Inter- and Intra-Domain Forwarding

As SCION is an inter-domain network architecture, it is not concerned with intra-domain forwarding. This corresponds to the general practice today where BGP and IP are used for inter-domain routing and forwarding, respectively, but ASes use an intra-domain protocol of their choice - for example OSPF or IS-IS for routing and IP, MPLS, and various Layer 2 protocols for forwarding. In fact, even if ASes use IP forwarding internally today, they typically encapsulate the original IP packet they receive at the edge of their network into another IP packet with the destination address set to the egress border router, to avoid full inter-domain forwarding tables at internal routers.

SCION emphasizes this separation as it is used exclusively for inter-domain forwarding; re-using the intra-domain network fabric to provide connectivity amongst all SCION infrastructure services, border routers, and endpoints. As a consequence, minimal change to the infrastructure is required for ISPs when deploying SCION.

Although a complete SCION address is composed of the <ISD, AS, endpoint address> 3-tuple, the endpoint address is not used for inter-domain routing or forwarding. This implies that the endpoint addresses are not required to be globally unique or globally routable and can be selected independently by the corresponding ASes. This means, for example, that an endpoint identified by a link-local IPv6 address in the source AS can directly communicate with an endpoint identified by a globally routable IPv4 address via SCION. It is possible for two SCION hosts with the same IPv4 address 10.0.0.42 but located in different ASes to communicate with each other via SCION ({{RFC1918}}).


### Intra-Domain Forwarding Process

The full forwarding process for a packet transiting an intermediate AS consists of the following steps.

**Note:** In this context, a border router is called an **ingress** border router when it refers to an entrance border router to an AS, as seen from the direction of travel of the SCION packet. A border router is called **egress** border router when it refers to an exit border router of an AS, as seen from the direction of travel of the SCION packet.

1. The AS's SCION ingress router receives a SCION packet from the neighboring AS.
2. The SCION router parses, validates, and authenticates the SCION header.
3. The SCION router maps the egress interface ID in the current Hop Field of the SCION header to the destination address of the intra-domain protocol (e.g. MPLS or IP) on the egress border router.
4. The packet is forwarded within the AS by routers and switches based on the header of the intra-domain protocol.
5. Upon receiving the packet, the SCION egress router strips off the header of the intra-domain protocol, again validates and updates the SCION header, and forwards the packet to the neighboring SCION router.
6. The last SCION router on the path forwards the packet to the packet's destination endpoint indicated by the field `DstHostAddr` of [the Address Header](#address-header).

### Configuration

Border routers require mappings from SCION interface IDs to underlay addresses and such information MUST be supplied to each router in an out of band fashion (e.g in a configuration file). For each link to a neighbor, these values MUST be configured. A typical implementation will require:

- Interface ID.
- Link type (core, parent, child, peer). Link type depends on mutual agreements between the organizations operating the ASes at each end of each link.
- Neighbor ISD-AS number.
- For the router that manages the interface: the neighbor interface underlay address.
- For the routers that do not manage the interface:  the address of the intra-domain protocol on the router that does.

In order to forward traffic to a service endpoint address (`DT/DS` == 0b01 in the [common header](#common-header)), a border router translates the service number into a specific destination address. The method used to accomplish the translation is not defined by this document and is only dependent on the implementation and the choices of each AS's administrator. In current practice this is accomplished by way of a configuration file.

**Note:** The current SCION implementation runs over the UDP/IP protocol. However, the use of other lower layers protocols is possible.


## Path Construction (Segment Combinations) {#construction}

Paths are discovered by the SCION control plane which makes them available to SCION endpoints in the form of path segments. As described in {{I-D.scion-cp}}, there are three kinds of path segments: up, down, and core. In the data plane, a SCION endpoint creates end-to-end paths from the path segments by combining multiple path segments. Depending on the network topology, a SCION forwarding path can consist of one, two, or three segments. Each path segment contains several Hop Fields representing the ASes on the segment as well as one Info Field with basic information about the segment, such as a timestamp.

Segments cannot be combined arbitrarily. To construct a valid forwarding path, the source endpoint MUST obey the following rules:

- There can be at most one of each type of segment (up, core, and down). Allowing multiple up or down segments would decrease efficiency and the ability of ASes to enforce path policies.
- If an up segment is present, it MUST be the first segment in the path.
- If a down segment is present, it MUST be the last segment in the path.
- If there are two path segments (one up and one down segment) that both announce the same peering link, then a shortcut via this peering link is possible.
- If there are two path segments (one up and one down segment) that share a common ancestor AS (in the direction of beaconing), then a shortcut via this common ancestor AS is possible.
- Additionally, all segments without any peering possibility MUST consist of at least two Hop Fields.

**Note:** The type of segment is known to the endpoint but is not visible in the path header of data packets. Therefore, a SCION router needs to explicitly verify that these rules were followed correctly.

Besides enabling the enforcement of path policies, the above rules also protect the economic interest of ASes as they prevent building "valley paths". A valley path contains ASes that do not profit economically from traffic on this route, with the name coming from the fact that such paths go "down" (following parent-child links) before going "up" (following child-parent links).

{{figure-1}} below shows valid segment combinations.

**Note:** It is assumed that the source and destination endpoints are in different ASes (as endpoints from the same AS use an empty forwarding path to communicate with each other).

~~~~
                                  ------- = end-to-end path
   C = Core AS                    - - - - = unused links
   * = source/destination AS      ------> = direction of beaconing


          Core                        Core                  Core
      ---------->                 ---------->           ---------->
     .-.       .-.               .-.       .-.         .-.       .-.
+-- ( C )-----( C ) --+     +-- ( C )-----(C/*)       (C/*)-----(C/*)
|    `+'       `+'    |     |    `+'       `-'         `-'       `-'
|     |    1a   |     |     |     |     1b                   1c
|     |         |     |     |     |
|     |         |     |     |     |
|    .+.       .+.    |     |    .+.                       Core
|   (   )     (   )   |     |   (   )                 -------------->
|    `+'       `+'    |     |    `+'                        .-.
|     |         |     |     |     |                   +----( C )----+
|     |         |     |     |     |                   |     `-'     |
|     |         |     |     |     |                   |             |
|    .+.       .+.    |     |    .+.                 .+.     1d    .+.
+-> ( * )     ( * ) <-+     +-> ( * )               (C/*)         (C/*)
     `-'       `-'               `-'                 `-'           `-'



          .-.                      .-.                   .-.
+--   +--( C )--+   --+      +--  (C/*)        +--    - ( C ) -    --+
|     |   `-'   |     |      |     `+'         |     |   `-'   |     |
|     |         |     |      |      |          |                     |
|     |    2a   |     |      |  2b  |          |     |    3a   |     |
|     |         |     |      |      |          |                     |
|    .+.       .+.    |      |     .+.         |    .+.       .+.    |
|   (   )     (   )   |      |    (   )        |   (   #-----#   )   |
|    `+'       `+'    |      |     `+'         |    `+'  Peer `+'    |
|     |         |     |      |      |          |     |         |     |
|     |         |     |      |      |          |     |         |     |
|     |         |     |      |      |          |     |         |     |
|    .+.       .+.    |      |     .+.         |    .+.       .+.    |
+-> ( * )     ( * ) <-+      +->  ( * )        +-> ( * )     ( * ) <-+
     `-'       `-'                 `-'              `-'       `-'


          Core                                    Core
      ---------->                              ---------->
     .-.       .-.             .-.            .-.       .-.        .-.
 +--( C )- - -( C )--+  +---- ( C ) ----+    ( C )- - -( C )  +-- ( C )
 |   `+'       `+'   |  |      `+'      |     `+'       `+'   |    `+'
 |         3b        |  |           4a  |          4b         |  5
 |    |         |    |  |       |       |      |         |    |     |
 |                   |  |               |                     |
 |   .+.       .+.   |  |      .+.      |      + - --. - +    |    .+.
 |  (   #-----#   )  |  |   +-(   )-+   |      +--(   )--+    |   ( * )
 |   `+'  Peer `+'   |  |   |  `-'  |   |      |   `-'   |    |    `+'
 |    |         |    |  |   |       |   |      |         |    |     |
 |    |         |    |  |   |       |   |      |         |    |     |
 |    |         |    |  |   |       |   |      |         |    |     |
 |   .+.       .+.   |  |  .+.     .+.  |     .+.       .+.   |    .+.
 +->( * )     ( * )<-+  +>( * )   ( * )<+    ( * )     ( * )  +-> ( * )
     `-'       `-'         `-'     `-'        `-'       `-'        `-'
~~~~
{: #figure-1 title="Illustration of valid path segment combinations. Each node represents a SCION Autonomous System."}


Valid path segment combinations:

- **Communication through core ASes**:

  - **Core-segment combination** (Cases 1a, 1b, 1c, 1d in {{figure-1}}): The up and down segments of source and destination do not have an AS in common. In this case, a core segment is REQUIRED to connect the source's up segment and the destination's down segment (Case 1a). If either the source or the destination AS is a core AS (Case 1b) or both are core ASes (Cases 1c and 1d), then no up or down segments are REQUIRED to connect the respective ASes to the core segment.
  - **Immediate combination** (Cases 2a, 2b in {{figure-1}}): The last AS on the up segment (which is necessarily a core AS) is the same as the first AS on the down segment. In this case, a simple combination of up and down segments creates a valid forwarding path. In Case 2b, only one segment is required.

- **Peering shortcut** (Cases 3a and 3b): A peering link exists between the up and down segment, and extraneous path segments to the core are cut off. Note that the up and down segments do not need to originate from the same core AS and the peering link could also be traversing to a different ISD.
- **AS shortcut** (Cases 4a and 4b): The up and down segments intersect at a non-core AS below the ISD core, thus creating a shortcut. In this case, a shorter path is made possible by removing the extraneous part of the path to the core. Note that the up and down segments do not need to originate from the same core AS and can even be in different ISDs (if the AS at the intersection is part of multiple ISDs).
- **On-path** (Case 5): In the case where the source's up segment contains the destination AS or the destination's down segment contains the source AS, a single segment is sufficient to construct a forwarding path. Again, no core AS is on the final path.


## Path Authorization

The SCION data plane provides *path authorization*. This property ensures that data packets always traverse the network using path segments that were explicitly authorized by the respective ASes and prevents endpoints from constructing unauthorized paths or paths containing loops. SCION uses symmetric cryptography in the form of Message Authentication Codes (MACs) to authenticate the information encoded in Hop Fields and such MACs are verified by routers at forwarding. For a detailed specification, see [](#path-auth).


# SCION Header Specification {#header}

This section provides a detailed specification of the SCION packet header. SCION also supports extension headers, which are additionally described.


## SCION Packet Header Format {#format}

The SCION packet header is composed of a common header, an address header, a path header, and an OPTIONAL extension header, see {{figure-2}} below.

~~~~
+--------------------------------------------------------+
|                     Common header                      |
|                                                        |
+--------------------------------------------------------+
|                     Address header                     |
|                                                        |
+--------------------------------------------------------+
|                      Path header                       |
|                                                        |
+--------------------------------------------------------+
|               Extension header (OPTIONAL)              |
|                                                        |
+--------------------------------------------------------+
~~~~
{: #figure-2 title="High-level SCION header structure"}

The *common header* contains important meta information like a version number and lengths of the header and payload. In particular, it contains flags that control the format of subsequent headers such as the address and path headers. For more details, see [](#common-header).

The *address header* contains the ISD, AS and endpoint addresses of source and destination. The type and length of endpoint addresses are variable and can be set independently using flags in the common header. For more details, see [](#address-header).

The *path header* contains the full AS-level forwarding path of the packet. A path type field in the common header specifies the path format used in the path header. For more details, see [](#path-header).

Finally, the OPTIONAL *extension* header contains a variable number of hop-by-hop and end-to-end options, similar to the extensions in the IPv6 header {{RFC8200}}. For more details, see [](#ext-header).


### Header Alignment

The SCION header is aligned to 4 bytes.


## SCION Header Structure

### Common Header {#common-header}

The SCION common header has the following packet format:

~~~~
0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  TraffCl      |                FlowID                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    NextHdr    |    HdrLen     |          PayloadLen           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    PathType   |DT |DL |ST |SL |              RSV              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-3 title="The SCION common header packet format"}

- `Version`: The version of the SCION common header. Currently, only version "0" is supported.
- `TrafficClass` (`TraffCl` in the image above): The 8-bit long identifier of the packet's class or priority. The value of the traffic class bits in a received packet or fragment might differ from the value sent by the packet's source. The current use of the `TrafficClass` field for Differentiated Services and Explicit Congestion Notification is specified in {{RFC2474}} and {{RFC3168}}.
- `FlowID`: This 20-bit field labels sequences of packets to be treated in the network as a single flow. Sources MUST set this field.
- `NextHdr`: Encodes the type of the first header after the SCION header. This can be either a SCION extension or a Layer 4 protocol such as TCP or UDP. Values of this field respect the Assigned SCION Protocol Numbers (see [](#protnum)).
- `HdrLen`: Specifies the entire length of the SCION header in bytes, i.e. the sum of the lengths of the common header, the address header, and the path header. All SCION header fields are aligned to a multiple of 4 bytes. The SCION header length is computed as `HdrLen` * 4 bytes. The 8 bits of the `HdrLen` field limit the SCION header to a maximum of 255 * 4 = 1020 bytes.
- `PayloadLen`: Specifies the length of the payload in bytes. The payload includes (SCION) extension headers and the L4 payload. This field is 16 bits long, supporting a maximum payload size of 65'535 bytes.
- `PathType`: Specifies the type of the SCION path. It is possible to specify up to 256 different types. The format of one path type is independent of all other path types. The currently defined SCION path types are Empty (0), SCION (1), OneHopPath (2), EPIC (3) and COLIBRI (4). This document only specifies the Empty, SCION and OneHopPath path types. The other path types are currently experimental.

| Value | Path Type                      |
|-------+--------------------------------|
| 0     | Empty path (`EmptyPath`)       |
| 1     | SCION (`SCION`)                |
| 2     | One-hop path (`OneHopPath`)    |
| 3     | EPIC path (experimental)       |
| 4     | COLIBRI path (experimental)    |
{: #table-1 title="SCION path types"}

- `DT/DL/ST/SL`: These fields define the endpoint address type and endpoint address length for the source and destination endpoint. `DT` and `DL` stand for Destination Type and Destination Length, whereas `ST` and `SL` stand for Source Type and Source Length. The possible endpoint address length values are 4 bytes, 8 bytes, 12 bytes, and 16 bytes. If some address has a length different from the supported values, the next larger size can be used and the address can be padded with zeros. {{table-2}} below lists the currently used values for address length. The "type" identifier is only defined in combination with a specific address length. For example, address type "0" is defined as IPv4 in combination with address length 4, but in combination with address length 16, it stands for IPv6. Per address length, several sub-types are possible. {{table-3}} shows the currently assigned combinations of lengths and types.

| DL/SL Value | Address Length |
|-------------+----------------|
| 0           | 4 bytes        |
| 1           | 8 bytes        |
| 2           | 12 bytes       |
| 3           | 16 bytes       |
{: #table-2 title="Address length values"}

| Length (bytes) | Type | DT/DL or ST/SL encoding | Conventional Use |
|----------------+------+-------------------------+------------------|
| 4              | 0    | 0b0000                  | IPv4             |
| 4              | 1    | 0b0100                  | Service          |
| 16             | 0    | 0b0011                  | IPv6             |
| other          |      |                         | Unassigned       |
{: #table-3 title="Allocations of type values to length values"}

A service address designates a set of endpoint addresses rather than a singular one. A packet addressed to a service is redirected to any one endpoint address that is known to be part of the set. {{table-4}} lists the known services.

- `RSV`: These bits are currently reserved for future use.

### Address Header {#address-header}

The SCION address header has the following format:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            DstISD             |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                             DstAS                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            SrcISD             |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                             SrcAS                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    DstHostAddr (variable Len)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    SrcHostAddr (variable Len)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-4 title="The SCION address header packet format"}

- `DstISD, SrcISD`: The 16-bit ISD identifier of the destination/source.
- `DstAS, SrcAS`: The 48-bit AS identifier of the destination/source.
- `DstHostAddr, SrcHostAddr`: Specifies the variable length endpoint address of the destination/source. The accepted type and length are defined in the `DT/DL/ST/SL` fields of the common header.

If a service address is implied by the `DT/DL` or `ST/SL` field of the common header, the corresponding address field has the following format:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Service Number        |              RSV              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-20 title="Service address format"}

- `RSV`: reserved for future use

The currently known service numbers are:

| Service Number (hex) | Short Name | Description            |
|-------------------+------------+------------------------|
| 0x0001            | DS         | Discovery Service      |
| 0x0002            | CS         | Control Service        |
| 0xFFFF            | None       | Reserved invalid value |
{: #table-4 title="Known Service Numbers"}


**Note:** For more information on addressing in SCION, see the introduction of the SCION Control Plane Specification ({{I-D.scion-cp}}).


### Path Header {#path-header}

The path header of a SCION packet differs for each SCION path type.

**Note:** The path type is set in the `PathType` field of the SCION common header.

Currently, SCION supports three path types:

- The `Empty` path type (`PathType=0`). For more information, see [](#empty).
- The `SCION` path type (`PathType=1`). This is the standard path type in SCION. For a detailed description, see [](#scion-path-type).
- The `OneHopPath` path type (`PathType=2`). For more information, see [](#onehop).


#### Empty Path Type {#empty}

The `Empty` path type is used to send traffic within an AS. It has no additional fields, i.e. it consumes 0 bytes on the wire.

One use case of the `Empty` path type lies in the context of link failure detection. To this end, SCION uses the Bidirectional Forwarding Detection (BFD) protocol ({{RFC5880}} and {{RFC5881}}). BFD is a protocol intended to detect faults in the bidirectional path between two forwarding engines with typically very low latency. It operates independently of media, data protocols, and routing protocols. SCION uses the `Empty` path type, together with `OneHopPath` path type, to bootstrap BFD within SCION. (For more information on the `OneHopPath` path type, see [](#onehop).)

One use case of the `Empty` path type lies in the context of [link-failure detection](#scion-bfd).

#### SCION Path Type {#scion-path-type}

This section specifies the standard `SCION` path type.
A SCION path has the following layout:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          PathMetaHdr                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           InfoField                           |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              ...                              |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           InfoField                           |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           HopField                            |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           HopField                            |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              ...                              |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-5 title="Layout of a standard SCION path"}


It consists of a path meta header, up to 3 Info Fields and up to 64 Hop Fields.

- The path meta header (`PathMetaHdr`) indicates the currently valid Info Field and Hop Field while the packet is traversing the network along the path, as well as the number of Hop Fields per segment.
- The number of Info Fields (`InfoField`) equals the number of path segments that the path contains - there is one Info Field per path segment. Each Info Field contains basic information about the corresponding segment, such as a timestamp indicating the creation time. There are also two flags: one specifies whether the segment is to be traversed in construction direction, the other whether the first or last Hop Field in the segment represents a peering Hop Field.
- Each Hop Field (`HopField`) represents a hop through an AS on the path, with the ingress and egress interface identifiers for this AS. This information is authenticated with a Message Authentication Code (MAC) to prevent forgery.

The SCION header is created by extracting the required Info Fields and Hop Fields from the corresponding path segments. The process of extracting is illustrated in {{figure-6}} below. Note that ASes at the joints of multiple segments are represented by two Hop Fields. Be aware that these Hop Fields are not equal! In the Hop Field that represents the last Hop in the first segment (seen in the direction of travel), only the ingress interface will be specified. However, in the hop Field that represents the first hop in the second segment (also in the direction of travel), only the egress interface will be defined. Thus, the two Hop Fields for this one AS build a full hop through the AS, specifying both the ingress and egress interface. As such, they bring the two adjacent segments together.

~~~~
                   +-----------------+
                   |    ISD Core     |
  .--.    .--.     |  .--.     .--.  |     .--.    .--.
 (AS 3)--(AS 4)----|-(AS 1)---(AS 2)-|----(AS 5)--(AS 6)
  `--'    `--'     |  `--'     `--'  |     `--'    `--'
                   +-----------------+

 Up-Segment           Core-Segment        Down-Segment
+---------+           +---------+         +---------+
| +-----+ |           | +-----+ |         | +-----+ |
| + INF + |--------+  | + INF + |---+     | + INF + |--+
| +-----+ |        |  | +-----+ |   |     | +-----+ |  |
| +-----+ |        |  | +-----+ |   |     | +-----+ |  |
| | HF  | |------+ |  | | HF  | |---+-+   | | HF  | |--+-+
| +-----+ |      | |  | +-----+ |   | |   | +-----+ |  | |
| +-----+ |      | |  | +-----+ |   | |   | +-----+ |  | |
| | HF  | |----+ | |  | | HF  | |---+-+-+ | | HF  | |--+-+-+
| +-----+ |    | | |  | +-----+ |   | | | | +-----+ |  | | |
| +-----+ |    | | |  +---------+   | | | | +-----+ |  | | |
| | HF  | |--+ | | |                | | | | | HF  | |--+-+-+-+
| +-----+ |  | | | |  +----------+  | | | | +-----+ |  | | | |
+---------+  | | | |  | ++++++++ |  | | | +---------+  | | | |
             | | | |  | | Meta | |  | | |              | | | |
             | | | |  | ++++++++ |  | | |              | | | |
             | | | |  | +------+ |  | | |              | | | |
             | | | +->| + INF  + |  | | |              | | | |
             | | |    | +------+ |  | | |              | | | |
             | | |    | +------+ |  | | |              | | | |
             | | |    | + INF  + |<-+ | |              | | | |
             | | |    | +------+ |    | |              | | | |
             | | |    | +------+ |    | |              | | | |
             | | |    | + INF  + |<---+-+--------------+ | | |
             | | |    | +------+ |    | |                | | |
             | | |    | +------+ |    | |                | | |
             | | +--->| |  HF  | |    | |                | | |
             | |      | +------+ |    | |                | | |
             | |      | +------+ |    | |                | | |
             | +----->| |  HF  | |    | |                | | |
             |        | +------+ |    | |                | | |
             |        | +------+ |    | |                | | |
             +------->| |  HF  | |    | |                | | |
                      | +------+ |    | |                | | |
                      | +------+ |    | |                | | |
                      | |  HF  | |<---+ |                | | |
                      | +------+ |      |                | | |
                      | +------+ |      |                | | |
     Forwarding Path  | |  HF  | |<-----+                | | |
                      | +------+ |                       | | |
                      | +------+ |                       | | |
                      | |  HF  | |<----------------------+ | |
                      | +------+ |                         | |
                      | +------+ |                         | |
                      | |  HF  | |<------------------------+ |
                      | +------+ |                           |
                      | +------+ |                           |
                      | |  HF  | |<--------------------------+
                      | +------+ |
                      +----------+
~~~~
{: #figure-6 title="Path construction example"}



##### Path Meta Header Field {#PathMetaHdr}

The 4-byte Path Meta Header field (`PathMetaHdr`) defines meta information about the SCION path that is contained in the path header. It has the following format:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| C |  CurrHF   |    RSV    |  Seg0Len  |  Seg1Len  |  Seg2Len  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-7 title="SCION path type - Format of the Path Meta Header field"}


- `C` (urrINF): Specifies a 2-bits index (0-based) pointing to the current Info Field for the packet on its way through the network. For details, see [](#offset-calc) below.
- `CurrHF`: Specifies a 6-bits index (0-based) pointing to the current Hop Field for the packet on its way through the network. For details, see [](#offset-calc) below. Note that the `CurrHF` index MUST point to a Hop Field that is part of the current path segment, as indicated by the `CurrINF` index.

Both indices are used by SCION routers when forwarding data traffic through the network. The SCION routers also increment the indexes if required. For more details, see [](#process-router).

- `Seg{0,1,2}Len`: The number of Hop Fields in a given segment. Seg{i}Len > 0 implies that segment *i* contains at least one Hop Field, which means that Info Field *i* exists. (If Seg{i}Len = 0 then segment *i* is empty, meaning that this path does not include segment *i*, and therefore there is no Info Field *i*.) The following rules apply:

  - The total number of Hop Fields in an end-to-end path MUST be equal to the sum of all `Seg{0,1,2}Len` contained in this end-to-end path.
  - It is an error to have Seg{X}Len > 0 AND Seg{Y}Len == 0, where 2 >= *X* > *Y* >= 0. That is, if path segment Y is empty, the following path segment X MUST also be empty.

- `RSV`: Unused and reserved for future use.


##### Path Offset Calculations {#offset-calc}

The path offset calculations are used by the SCION border routers to determine the Info Field and Hop Field that are currently valid for the packet on its way through the network.

The following rules apply when calculating the path offsets:

~~~~
   if Seg2Len > 0: NumINF = 3
   else if Seg1Len > 0: NumINF = 2
   else if Seg0Len > 0: NumINF = 1
   else: invalid
~~~~

The offsets of the current Info Field and current Hop Field (relative to the end of the address header) are now calculated as:

~~~~
   B = byte
   InfoFieldOffset = 4B + 8B.CurrINF
   HopFieldOffset = 4B + 8B.NumINF + 12B.CurrHF
~~~~

To check that the current Hop Field is in the segment of the current Info Field, the `CurrHF` needs to be compared to the `SegLen` fields of the current and preceding Info Fields.


##### Info Field {#inffield}

The 8-byte Info Field (`InfoField`) has the following format:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|r r r r r r P C|      RSV      |             Acc               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-8 title="SCION path type - Format of the Info Field"}


- `r`: The `r` bits are unused and reserved for future use.
- `P`: Peering flag. If the flag has value "1", the segment represented by this Info Field contains a peering Hop Field, which requires special processing in the data plane. For more details, see [](#peerlink) and [](#packet-verif).
- `C`: Construction direction flag. If the flag has value "1", the Hop Fields in the segment represented by this Info Field are arranged in the direction they have been constructed during beaconing.
- `RSV`: Unused and reserved for future use.
- `Acc`: This updatable field/counter is REQUIRED for calculating the MAC in the data plane. `Acc` stands for "Accumulator". For more details, see [](#auth-chained-macs).
- `Timestamp`: Timestamp created by the initiator of the corresponding beacon. The timestamp is defined as the number of seconds elapsed since the POSIX Epoch (1970-01-01 00:00:00 UTC), encoded as a 32-bit unsigned integer. This timestamp enables the validation of a Hop Field in the segment represented by this Info Field, by verifying the expiration time and MAC set in the Hop Field - the expiration time of a Hop Field is calculated relative to the timestamp. A Info field with a timestamp in the future is invalid. For the purpose of validation, a timestamp is considered "future" if it is later than the locally available current time plus 337.5 seconds (i.e. the minimum time to live of a hop).


##### Hop Field {#hopfld}

The 12-byte Hop Field (``HopField``) has the following format:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|r r r r r r I E|    ExpTime    |           ConsIngress         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        ConsEgress             |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                              MAC                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-9 title="SCION path type - Format of the Hop Field"}


- `r`: The `r` bits are unused and reserved for future use.
- `I`: The ConsIngress Router Alert flag. If the ConsIngress Router Alert flag has value "1", the ingress router (in construction direction) will treat the payload of the packet as a message to itself and attempt to process it according to the value of the `NextHdr` field. Such messages include [Traceroute Requests](#traceroute-request).
- `E`: The ConsEgress Router Alert flag. If the ConsEgress Router Alert flag has value "1", the egress router (in construction direction) will treat the payload of the packet as a message to itself and attempt to process it according to the value of the `NextHdr` field. Such messages include [Traceroute Requests](#traceroute-request).
- `ExpTime`: Expiration time of a Hop Field. The field is 1-byte long, thus there are 256 different values available to express an expiration time. The expiration time specified in this field is relative. An absolute expiration time in seconds is computed in combination with the `Timestamp` field (from the corresponding Info Field), as follows:

  - `Timestamp` + (1 + `ExpTime`) * (24 hours/256)

- `ConsIngress`, `ConsEgress`: The 16-bits ingress/egress interface IDs in construction direction, that is, the direction of beaconing.
- `MAC`: The 6-byte Message Authentication Code to authenticate the Hop Field. For details on how this MAC is calculated, see [](#hf-mac-calc).

The two Router Alert flags are commonly named `ConsIngress` and `ConsEgress` as a reminder that they request an action by the router owning the Ingress or Egress interfaces when the packet is traveling in the *construction direction* of the path segment (i.e. the direction of beaconing). When the packet is traveling in the opposite direction the meanings are reversed. These *flags* (i.e. single bit) are not to be confused with the `ConsIngress` and `ConsEgress` *fields* which designate the interfaces for the purpose of forwarding the packet and are always set, regardless of any router alert.

A sender cannot rely on multiple routers retrieving and processing the payload even if it sets multiple router alert flags. This is use-case dependent: In the case of Traceroute informational messages, for example, the router for which the traceroute request is intended will process the request (if the corresponding Router Alert flag is set to "1") and reply to it without further forwarding the request along the path. Use cases that require multiple routers/hops on the path to process a packet have to instead rely on a hop-by-hop extension (see [](#ext-header)). For general information on router alerts, see {{RFC2711}}.


#### One-Hop Path Type {#onehop}

The one-hop path type `OneHopPath` is currently used to bootstrap beaconing between neighboring ASes. This is necessary, as neighbor ASes do not have a forwarding path yet before beaconing is started.

A one-hop path has exactly one Info Field and two Hop Fields with the specialty that the second Hop Field is not known a priori, but is instead created by the ingress SCION border router of the neighboring AS while processing the one-hop path. Any entity with access to the forwarding key of the source endpoint AS can create a valid info and Hop Field as described in [](#inffield) and [](#hopfld), respectively.

Upon receiving a packet containing a one-hop path, the ingress border router of the destination AS fills in the `ConsIngress` field in the second Hop Field of the one-hop path with the ingress interface ID. It sets the `ConsEgress` field to an invalid value (e.g. unspecified value 0), ensuring the path cannot be used beyond the destination AS. Then it calculates and appends the appropriate MAC for the Hop Field.

{{figure-10}} below shows the layout of a SCION one-hop path type. There is only a single Info Field; the appropriate Hop Field can be processed by a border router based on the source and destination address. In this context, the following rules apply:

- At the source endpoint AS, *CurrHF := 0*.
- At the destination endpoint AS, *CurrHF := 1*.

~~~~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           InfoField                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           HopField                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           HopField                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-10 title="Layout of the SCION one-hop path type"}


#### Path Reversal {#reverse}

When a destination endpoint receives a SCION packet, it can use the path information in the SCION header for sending the reply packets. For this, the destination endpoint MUST perform the following steps:

1. Reverse the order of the Info Fields;
2. Reverse the order of the Hop Fields;
3. For each Info Field, negate the construction direction flag `C`; do not change the accumulator field `Acc`.
4. In the `PathMetaHdr` field:

   - Set the `CurrINF` and `CurrHF` to "0".
   - Reverse the order of the non-zero `SegLen` fields.

   **Note:** For more information on the `PathMetaHdr` field, see [](#PathMetaHdr).


## Extension Headers {#ext-header}

This section specifies the SCION extension headers. SCION currently provides two types of extension headers: the Hop-by-Hop (HBH) Options header and the End-to-End (E2E) Options header.

- The Hop-by-Hop Options header is used to carry OPTIONAL information that MAY be examined and processed by every SCION router along a packet's delivery path. The Hop-by-Hop Options header is identified by value "200" in the `NextHdr` field of the SCION common header (see [](#common-header)).
- The End-to-End Options header is used to carry OPTIONAL information that MAY be examined and processed by the sender and/or the receiver of the packet. The End-to-End Options header is identified by value "201" in the `NextHdr` field of the SCION common header (see [](#common-header)).

If both headers are present, the HBH Options header MUST come before the E2E Options header.

**Note:** The SCION extension headers are defined and used based on and similar to the IPv6 extensions as specified in section 4 of {{RFC8200}}. The SCION Hop-by-Hop Options header and End-to-End Options header resemble the IPv6 Hop-by-Hop Options Header (section 4.3 in the RFC) and Destination Options Header (section 4.6), respectively.


### Format of the SCION Options Headers {#oh-format}

The SCION Hop-by-Hop Options and End-to-End Options headers have the following format:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    NextHdr    |     ExtLen    |            Options            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-11 title="Extension headers: Options header"}


- `NextHdr`: Unsigned 8-bit integer. Identifies the type of header immediately following the Hop-by-Hop/End-to-End Options header. Values of this field respect the Assigned SCION Protocol Numbers (see also [](#protnum)).
- `ExtLen`: 8-bit unsigned integer. The length of the Hop-by-hop or End-to-end options header in 4-octet units, not including the first 4 octets. That is: `ExtLen = uint8(((L + 2) / 4) - 1)`, where `L` is the size of the header in bytes, assuming that `L + 2` is a multiple of 4.
- `Options`: This is a variable-length field. The length of this field MUST be such that the complete length of the Hop-by-Hop/End-to-End Options header is an integer multiple of 4 bytes. This can be achieved by using options of type 0 or 1 (see {{table-4}}) . The `Options` field contains one or more Type-Length-Value (TLV) encoded options. For details, see [](#optfld).

The Hop-by-Hop/End-to-End Options header is aligned to 4 bytes.


#### Options Field {#optfld}

The `Options` field of the Hop-by-Hop Options and the End-to-End Options headers carries a variable number of options that are type-length-value (TLV) encoded. Each TLV-encoded option has the following format:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    OptType    |  OptDataLen   |            OptData            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                              ...                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-12 title="Options field: TLV-encoded options"}


- `OptType`: 8-bit identifier of the type of option. The following option types are assigned to the SCION HBH/E2E Options header:

| Decimal | Option Type                                                |
|---------+------------------------------------------------------------|
| 0       | Pad1 (see [](#pad1))                                       |
| 1       | PadN (see [](#padn))                                       |
| 2       | SCION Packet Authenticator Option.<br> Only used by the End-to-End Options header (experimental). |
| 253     | Used for experimentation and testing                       |
| 254     | Used for experimentation and testing                       |
| 255     | Reserved                                                   |
{: #table-5 title="Option types of SCION Options header"}

- `OptDataLen`: Unsigned 8-bit integer denoting the length of the `OptData` field of this option in bytes.
- `OptData`: Variable-length field. Option-type specific data.

The options within a header MUST be processed strictly in the order they appear in the header. This is to prevent a receiver from, for example, scanning through the header looking for a specific option and processing this option prior to all preceding ones.

Individual options may have specific alignment requirements, to ensure that multibyte values within the `OptData` fields have natural boundaries. The alignment requirement of an option is specified using the notation "xn+y". This means that the `OptType` MUST appear at an integer multiple of x bytes from the start of the header, plus y bytes. For example:

- `2n`: means any 2-bytes offset from the start of the header.
- `4n+2`: means any 4-bytes offset from the start of the header, plus 2 bytes.

There are two padding options to align subsequent options and to pad out the containing header to a multiple of 4 bytes in length - for details, see below. All SCION implementations MUST recognize these padding options.



##### Pad1 Option {#pad1}

Alignment requirement: none.

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|       0       |
+-+-+-+-+-+-+-+-+
~~~~
{: #figure-13 title="TLV-encoded options - Pad1 option"}


**Note:** The format of the Pad1 option is a special case - it does not have length and value fields.

The Pad1 option is used to insert 1 byte of padding into the `Options` field of an extension header. If more than one byte of padding is required, the PadN option SHOULD be used, rather than multiple Pad1 options. See the next section for more information on the PadN option.


##### PadN Option {#padn}

Alignment requirement: none.

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       1       |  OptDataLen   |            OptData            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                              ...                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-14 title="TLV-encoded options - PadN option"}


The PadN option is used to insert two or more bytes of padding into the `Options` field of an extension header. For N bytes of padding, the `OptDataLen` field contains the value N-2, and the `OptData` consists of N-2 zero-valued bytes.


## Upper-Layer Protocol Issues


### Pseudo Header for Upper-Layer Checksum {#pseudo}

Any transport or other upper-layer protocol that includes addresses from the SCION header in the checksum computation MUST use the following pseudo header:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ -----+
|            DstISD             |                               |      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               + SCION|
|                             DstAS                             |   ad-|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ dress|
|            SrcISD             |                               |  hea-|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +   der|
|                             SrcAS                             |      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+      |
|                    DstHostAddr (variable Len)                 |      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+      |
|                    SrcHostAddr (variable Len)                 |      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ -----+
|                    Upper-Layer Packet Length                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      zero                     |  Next Header  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-15 title="Layout of the pseudo header for the upper-layer checksum"}


- `DstISD`, `SrcISD`, `DstAS`, `SrcAS`, `DstHostAddr`, `SrcHostAddr`: These values are taken from the SCION address header.
- `Upper-Layer Packet Length`: The length of the upper-layer header and data. Some upper-layer protocols define headers that carry the length information explicitly (e.g., UDP). This information is used as the upper-layer packet length in the pseudo header for these protocols. The remaining protocols, which do not carry the length information directly, use the value from the `PayloadLen` field in the SCION common header, minus the sum of the extension header lengths.
- `Next Header`: The protocol identifier associated with the upper-layer protocol (e.g., 17 for UDP - see also [](#protnum)). This field can differ from the `NextHdr` field in the SCION common header, if extensions are present.



# Life of a SCION Data Packet

This section gives a high-level description of the life cycle of a SCION packet: How it is crafted at its source endpoint, passes through a number of routers, and finally reaches its destination endpoint. It is assumed that both source and destination are native SCION endpoints (i.e., they both run a native SCION network stack).

To keep it brief, the example illustrates an intra-ISD case, i.e., all communication happens within a single ISD. As the sample ISD only consists of one core AS, the end-to-end path only includes an up-path and down-path segment. In the case of inter-ISD forwarding, the complete end-to-end path from source endpoint to destination endpoint would always require a core-path segment, too. But this makes no difference for the forwarding process, which works the same in an intra- and inter-ISD context. We therefore abstain from describing the inter-ISD forwarding.


## Description

~~~~
                    +--------------------+
                    |                    |
                    |        AS 1        |
                    |                    |
                    |                    |
                    |     198.51.100.4 .-+. i1b (1-1,198.51.100.17)
                    |          +------( R3 )---+
                   .+-.        |       `-+'    |
          +-------( R2 )-------+         |     |
          |    i1a `+-' 198.51.100.1     |     |
          |         |                    |     |
          |         +--------------------+     | (1-3,198.51.100.18)
          |                                    | i3a
          |                                   .+-.
    i2a .-+.                          +------( R4 )--------+
+------( R1 )--------+                |       `-+'         |
|       `-+'         |                |         |192.0.2.34|
|         |203.0.113.17               |         |          |
|         |          |                |         |    AS 3  |
|         |    AS 2  |                |         |          |
|         |          |                |       +---+        |
|       +---+        |                |       | B |        |
|       | A |        |                |       +---+        |
|       +---+        |                |   1-3,192.0.2.7    |
|  1-2,203.0.113.6   |                |                    |
|                    |                +--------------------+
+--------------------+
~~~~
{: #figure-16 title="Sample topology to illustrate the life cycle of a SCION packet. AS 1 is the core AS of ISD 1, and AS 2 and AS 3 are non-core ASes of ISD 1."}


Based on the network topology in {{figure-16}} above, this example shows the path of a SCION packet sent from source endpoint A to destination endpoint B, and how it will be processed by each router on the path. This is done by means of simplified snapshots of the packet header after each such processing step. These snapshots, which are depicted in tables, show the most relevant information of the header, i.e., the SCION path and IP encapsulation for local communication.


## Creating an End-to-End SCION Forwarding Path

In this example, source endpoint A in AS 2 wants to send a data packet to destination endpoint B in AS 3. Both AS 2 and AS 3 are part of ISD 1. To create an end-to-end SCION forwarding path, source endpoint A first requests its own AS-2 control service for up-segments to the core AS in its ISD. The AS-2 control service will return up-segments from AS 2 to the ISD core. Endpoint A will also query its AS-2 control service for a down-segment from its ISD core AS to AS 3, in which destination endpoint B is located. The AS-2 control service (possibly after connecting to the core control service) will return down-segments from the ISD core down to AS 3.

**Note:** For more details on the lookup of path segments, see the section "Path Lookup" in the Control Plane specification ({{I-D.scion-cp}}).

Based on its own selection criteria, source endpoint A selects the up-segment (0,i2a)(i1a,0) and the down-segment (0,i1b)(i3a,0) from the path segments returned by its own AS-2 control service. The path segments consist of Hop Fields that carry the ingress and egress interfaces of each AS (e.g., i2a, i1a, ...), as described in detail in [](#format) - (x,y) represents one Hop Field.

To obtain an end-to-end forwarding path from the source AS to the destination AS, source endpoint A combines the two path segments into the resulting SCION forwarding path, which contains the two Info Fields *IF1* and *IF2* and the Hop Fields (0,i2a), (i1a,0), (0,i1b), and (i3a,0).

**Note:** As this brief sample path does not contain a core-segment, the end-to-end path only consists of two path segments.

Source endpoint A now adds this end-to-end forwarding path to the header of the packet that A wants to send to destination endpoint B, and starts transferring the packet. The following section describes what happens with the SCION packet header on the packet's way from A to B.


## Step-by-Step Explanation

This section explains the packet header modifications at each router, based on the network topology in {{figure-16}} above. Each step includes a table that represents a simplified snapshot of the packet header at the end of this specific step. Regarding the notation used in the figure/tables, each SRC and DST entry should be read as router (or endpoint) followed by its address. The current Info Field (with metadata on the current path segment) in the SCION header is depicted italic/cursive in the tables. The current Hop Field, representing the current AS, is shown bold. The snapshot tables also include references to IP/UDP addresses.

**Note:** In this context, a border router is called **ingress** border router when it refers to an entrance border router to an AS, as seen from the direction of travel of the SCION packet. So in the context here, the ingress border router is the *(packet) incoming* border router. A border router is called **egress** border router when it refers to an exit border router of an AS, as seen from the direction of travel of the SCION packet. So in this context, the egress border router is the *(packet) leaving* border router.


- *Step 1* <br> **A->R1**: The SCION-enabled source endpoint A in AS 2 creates a new SCION packet destined for destination endpoint B in AS 3, with payload P. Endpoint A sends the packet (for the chosen forwarding path) to the next SCION router as provided by its control service, which is in this case R1. A encapsulates the SCION packet into an underlay UDP/IPv4 header for the local delivery to R1, utilizing AS 2's internal routing protocol. The current Info Field is *IF1*. Upon receiving the packet, R1 will forward the packet on the egress interface that endpoint A has included into the first Hop Field of the SCION header.

|  A -> R1                                                     |
|------------+-------------------------------------------------|
| SCION      | SRC = 1-2,203.0.113.6 (source endpoint A) <br>  |
|            | DST = 1-3,192.0.2.7 (dest. endpoint B) <br>     |
|            | PATH = <br>                                     |
|            | - *IF1* **(0,i2a)** (i1a,0) <br>                |
|            | - IF2 (0,i1b) (i3a,0) <br>                      |
| UDP        | P<sub>S</sub> = 30041, P<sub>D</sub> = 30041 <br>   |
| IP         | SRC = 203.0.113.6 (endpoint A) <br>             |
|            | DST = 203.0.113.17 (router R1) <br>             |
| Link layer | SRC=A, DST=R1                                   |
{: title="Snapshot header - step 1"}


- *Step 2* <br> **R1->R2**: Router R1 inspects the SCION header and considers the relevant Info Field of the specified SCION path, which is the Info Field indicated by the current Info Field pointer. In this case, it is the first Info Field *IF1*. The current Hop Field is the first Hop Field (0,i2a), which instructs router R1 to forward the packet on its interface i2a. After reading the current Hop Field, R1 moves the pointer forward by one position to the second Hop Field (i1a,0). Note that, at this point, no underlay IP header is necessary, since the routers R1 and R2 are directly connected over layer 2.

  **Note:** Although technically there is no need for a UDP/IP underlay if two routers are directly connected, the SCION implementation always uses a UDP/IP underlay in practice. This is to enable a common interface for all routers.

|  R1 -> R2                                                    |
|------------+-------------------------------------------------|
| SCION      | SRC = 1-2,203.0.113.6 (source endpoint A) <br>  |
|            | DST = 1-3,192.0.2.7 (dest. endpoint B) <br>     |
|            | PATH = <br>                                     |
|            | - *IF1* (0,i2a) **(i1a,0)**  <br>               |
|            | - IF2 (0,i1b) (i3a,0) <br>                      |
| Link layer | SRC=R1, DST=R2                                  |
{: title="Snapshot header - step 2"}


- *Step 3* <br> **R2->R3**: When receiving the packet, router R2 of core AS 1 checks whether the packet has been received through the ingress interface i1a as specified by the current Hop Field. Otherwise, the packet is dropped by router R2. The router notices that it has consumed the last Hop Field of the current path segment, and hence moves the pointer from the current Info Field to the next Info Field *IF2*. The corresponding current Hop Field is (0,i1b), which contains egress interface i1b. R2 maps the i1b interface ID to egress router R3, it therefore encapsulates the SCION packet inside an intra-AS underlay IP packet with the address of R3 as the underlay destination.

|  R2 -> R3                                                   |
|------------+------------------------------------------------|
| SCION      | SRC = 1-2,203.0.113.6 (source endpoint A) <br> |
|            | DST = 1-3,192.0.2.7 (dest. endpoint B) <br>    |
|            | PATH =  <br>                                   |
|            | - IF1 (0,i2a) (i1a,0) <br>                     |
|            | - *IF2* **(0,i1b)** (i3a,0) <br>               |
| UDP        | P<sub>S</sub> = 30041, P<sub>D</sub> = 30041 <br> |
| IP         | SRC = 198.51.100.1 (router R2) <br>            |
|            | DST = 198.51.100.4 (router R3) <br>            |
| Link layer | SRC=R2, DST=R3                                 |
{: title="Snapshot header - step 3"}


- *Step 4* <br> **R3->R4**: Router R3 inspects the current Hop Field in the SCION header, uses interface i1b to forward the packet to its neighbor SCION router R4 of AS 3, and moves the current hop-field pointer forward. It adds an IP header to reach R4.


|  R3 -> R4                                                   |
|------------+------------------------------------------------|
| SCION      | SRC = 1-2,203.0.113.6 (source endpoint A) <br> |
|            | DST = 1-3,192.0.2.7 (dest. endpoint B) <br>    |
|            | PATH =  <br>                                   |
|            | - IF1 (0,i2a) (i1a,0) <br>                     |
|            | - *IF2* (0,i1b) **(i3a,0)** <br>               |
| UDP        | P<sub>S</sub> = 30041, P<sub>D</sub> = 30041 <br> |
| IP         | SRC = 1-1,198.51.100.17 (router R3) <br>       |
|            | DST = 1-3,198.51.100.18 (router R4) <br>       |
| Link layer | SRC=R3, DST=R4                                 |
{: title="Snapshot header - step 4"}


- *Step 5* <br> **R4->B**: SCION router R4 first checks whether the packet has been received through the ingress interface i3a as specified by the current Hop Field. R4 will then also realize, based on the fields `CurrHF` and `SegLen` in the SCION header, that the packet has reached the last hop in its SCION path. Therefore, instead of stepping up the pointers to the next info or Hop Field, router R4 inspects the SCION destination address and extracts the endpoint address 192.0.2.7. It creates a fresh underlay UDP/IP header with this address as destination and with itself as source. The intra-domain forwarding can now deliver the packet to destination endpoint B.

|  R4 -> B                                                    |
|------------+------------------------------------------------|
| SCION      | SRC = 1-2,203.0.113.6 (source endpoint A) <br> |
|            | DST = 1-3,192.0.2.7 (dest. endpoint B) <br>    |
|            | PATH =  <br>                                   |
|            | - IF1 (0,i2a) (i1a,0) <br>                     |
|            | - *IF2* (0,i1b) **(i3a,0)** <br>               |
| UDP        | P<sub>S</sub> = 30041, P<sub>D</sub> = 30041 <br> |
| IP         | SRC = 192.0.2.34 (router R4) <br>              |
|            | DST = 192.0.2.7 (endpoint B) <br>              |
| Link layer | SRC=R4, DST=B                                  |
{: title="Snapshot header - step 5"}


When destination endpoint B wants to respond to source endpoint A, it can just swap the source and destination addresses in the SCION header, reverse the SCION path, and set the pointers to the info and Hop Fields at the beginning of the reversed path (see also [](#reverse)).


# Path Authorization {#path-auth}

Path authorization guarantees that data packets always traverse the network along paths segments authorized by all on-path ASes in the control plane. In contrast to the IP-based Internet, where forwarding decisions are made by routers based on locally stored information, SCION routers base their forwarding decisions purely on the forwarding information carried in the packet header and set by endpoints.

SCION uses cryptographic mechanisms to efficiently provide path authorization. The mechanisms are based on *symmetric* cryptography in the form of Message Authentication Codes (MACs) in the data plane to secure forwarding information encoded in Hop Fields. This chapter first explains how Hop Field MACs are computed, then how they are validated as they traverse the network.


## Authorizing Segments through Chained MACs {#auth-chained-macs}

When authorizing SCION PCBs and path segments in the control plane and forwarding information in the data plane, an AS authenticates not only its own hop information but also an aggregation of all upstream hops. This section describes how this works.


### Hop Field MAC Computation {#hf-mac-calc}

The MAC in the Hop Fields of a SCION path has two purposes:

- Preventing malicious endpoints from illegally adding, removing, or reordering hops within a path segment created during beaconing in the control plane.
  In particular, preventing path splicing, i.e. the combination of parts of different valid path segments into a new, unauthorized, path segment.
- Authentication of the information contained in the Hop Field itself, in particular the `ExpTime`, `ConsIngress`, and `ConsEgress`.

To fulfill the above purposes, the MAC for the Hop Field of AS<sub>i</sub> includes both the components of the current Hop Field HF<sub>i</sub> and an aggregation of the path segment identifier and all preceding Hop Fields/entries in the path segment. The aggregation is a 16-bit XOR-sum of the path segment identifier and the Hop Field MACs.

When originating a path-segment construction beacon PCB in the **control plane**, a core AS chooses a random 16-bit value as segment identifier `SegID` for the path segment and includes it in the PCB's `Segment Info` component. In the control plane, each AS<sub>i</sub> on the path segment computes the MAC for the current hop HF<sub>i</sub>, based on the value of `SegID` and the MACs of the preceding hop entries. Here, the full XOR-sum is computed explicitly.

For high-speed packet processing in the **data plane**, computing even cheap operations such as the XOR-sum over a variable number of inputs is complicated, in particular for hardware router implementations. To avoid this overhead for the MAC-chaining in path authorization in the data plane, the XOR-sum is tracked incrementally for each (of the up to three) path segments in a path, as a separate, updatable accumulator field `Acc`. The routers update the accumulator field `Acc` by adding/subtracting only a single 16-bit value each.

When combining path segments to create a path to the destination endpoint, the source endpoint MUST also initialize the value of accumulator field `Acc` for each path segment. The `Acc` field MUST contain the correct XOR-sum of the path segment identifier and preceding Hop Field MACs expected by the first router that is traversed.

The aggregated 16-bit path segment identifier and preceding MACs prevents the splicing parts of different path segments, unless there is a per-chance collision of the `Acc` value among compatible path segments in on AS. See {{path-splicing}} for more details.

In the following, the computation of the Hop Field MAC as well as the accumulator field `Acc` is explained.

#### Hop Field MAC - Definition {#def-mac}

- Consider a path segment with "n" hops, containing ASes AS<sub>0</sub>, ... , AS<sub>n-1</sub>, with forwarding keys K<sub>0</sub>, ... , K<sub>n-1</sub> in this order.
- AS<sub>0</sub> is the core AS that created the PCB representing the path segment and that added a random initial 16-bit segment identifier `SegID` to the `Segment Info` field of the PCB.

The MAC<sub>i</sub> of the Hop Field of AS<sub>i</sub> is now calculated as:

MAC<sub>i</sub> = <br> Ck<sub>i</sub> (`SegID` XOR MAC<sub>0</sub> \[:2] ... XOR MAC<sub>i-1</sub> \[:2], Timestamp, ExpTime<sub>i</sub>, ConsIngress<sub>i</sub>, ConsEgress<sub>i</sub>)

where

- k<sub>i</sub> = The forwarding key k of the current AS<sub>i</sub>
- Ck<sub>i</sub> (...) = Cryptographic checksum C over (...) computed with forwarding key k<sub>i</sub>
- `SegID` = The random initial 16-bit segment identifier set by the core AS when creating the corresponding PCB
- XOR = The bitwise "exclusive or" operation
- MAC<sub>i</sub> \[:2] = The Hop Field MAC for AS<sub>i</sub>, truncated to 2 bytes
- Timestamp = The timestamp set by the core AS when creating the corresponding PCB
- ExpTime<sub>i</sub>, ConsIngress<sub>i</sub>, ConsEgress<sub>i</sub> = The content of the Hop Field HF<sub>i</sub>

Thus, the current MAC is based on the XOR-sum of the truncated MACs of all preceding Hop Fields in the path segment as well as the path segment's `SegID`. In other words, the current MAC is *chained* to all preceding MACs.
In order to effectively prevent path-splicing, the cryptographic checksum function used MUST ensure that the truncation of the MACs is non-degenerate and roughly uniformly distributed (see {{mac-requirements}}).

#### Accumulator Acc - Definition {#def-acc}

The accumulator Acc<sub>i</sub> is an updatable counter introduced in the data plane to avoid the overhead caused by MAC-chaining for path authorization. This is achieved by incrementally tracking the XOR-sum of the previous MACs as a separate, updatable accumulator field `Acc`, which is part of the path segment's Info Field `InfoField` in the packet header (see also [](#inffield)). Routers update this field by adding/subtracting only a single 16-bit value each.

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|r r r r r r P C|      RSV      |             Acc               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-17 title="The Info Field of a specific path segment in the packet header, with the updatable accumulator field `Acc`."}


This is how it works:

[](#def-mac) defines MAC<sub>i</sub> as follows:

MAC<sub>i</sub> = <br> Ck<sub>i</sub> (`SegID` XOR MAC<sub>0</sub> \[:2] ... XOR MAC<sub>i-1</sub> \[:2], Timestamp, ExpTime<sub>i</sub>, ConsIngress<sub>i</sub>, ConsEgress<sub>i</sub>)

In the data plane, the expression `SegID XOR MAC_0 [:2] ... XOR MAC_i-1 [:2]` is replaced by Acc<sub>i</sub>. This results in the following alternative procedure for the computation of MAC<sub>i</sub> used in the data plane:

MAC<sub>i</sub> = Ck<sub>i</sub> (Acc<sub>i</sub>, Timestamp, ExpTime<sub>i</sub>, ConsIngress<sub>i</sub>, ConsEgress<sub>i</sub>)

During forwarding in the data plane, each AS<sub>i</sub> updates the `Acc` field in the packet header, such, that it contains the correct input value of the Accumulator Acc for the next AS in the path segment to be able to calculate the MAC over its Hop Field. Note that the correct input value of the `Acc` field depends on the direction of travel.

The value of the accumulator Acc<sub>i+1</sub> is calculated based on the following definition (in the direction of beaconing):

Acc<sub>i+1</sub> = Acc<sub>i</sub> XOR MAC<sub>i</sub> \[:2]

- XOR = The bitwise "exclusive or" operation
- MAC<sub>i</sub> \[:2] = The Hop Field MAC for the current AS<sub>i</sub>, truncated to 2 bytes


#### Default Hop Field MAC Algorithm

The algorithm used to compute the Hop Field MAC is an AS-specific choice. The operator of an AS can freely choose any MAC algorithm without outside coordination. However, the control service and routers of the AS do need to agree on the algorithm used.
All control service and router implementations MUST support the Default Hop Field MAC algorithm described below.

The default MAC algorithm is AES-CMAC ({{RFC4493}}) truncated to 48-bits, computed over the Info Field and the first 6 bytes of the Hop Field, with flags and reserved fields zeroed out. The input is padded to 16 bytes. The _first_ 6 bytes of the AES-CMAC output are used as resulting Hop Field MAC.
{{figure-18}} below shows the layout of the input data to calculate the Hop Field MAC.

~~~~
0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ -----+
|               0               |           Acc                 |  Info|
|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-| Field|
|                           Timestamp                           |      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-| -----+
|       0       |    ExpTime    |          ConsIngress          |   Hop|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ Field|
|          ConsEgress           |               0               |      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ -----+
~~~~
{: #figure-18 title="Input data to calculate the Hop Field MAC for the default hop-field MAC algorithm"}


#### Alternative Hop Field MAC Algorithms {#mac-requirements}

For alternative algorithms, the following requirements MUST all be met:
- The Hop Field MAC field is computed as a function of the secret forwarding key, the `Acc`, `Timestamp` fields of the Info Field and the `ExpTime`, `ConsIngress` and `ConsEgress` fields of the Hop Field.
  Function is used in the mathematical sense, that is, for any values of these inputs there is exactly one result.
- The algorithm returns an unforgable 48-bit value.
  Unforgable specifically means "existentially unforgable under a chosen message attack" ({{CRYPTOBOOK}}). Informally, this means an attacker without access to the secret key has no computationally efficient means to create a valid MAC for some attacker chosen input values, even if it has access to an "oracle" providing a valid MAC for any other input values.
- The truncation of the result value to the first 2 bytes / 16 bits of the result value:
    - is not degenerate, i.e. any small change in any input value SHOULD have an "avalanche effect" on these bits, and
    - is roughly uniformly distributed when considering all possible input values.

  This additional requirment is naturally satisfied for MAC algorithms based on typical block ciphers or hash algorithms.
  It ensures that the MAC chaining via the `Acc` field is not degenerate.

### Peering Links {#peerlink}


The above described computation of a Hop Field MAC does not apply to a peering Hop Field, i.e., to a Hop Field that allows transiting from a child interface/link to a peering interface/link.

The reason for this is that the MACs of the Hop Fields "after" the peering Hop Field (in beaconing direction) are not chained to the MAC of the peering Hop Field, but to the MAC of the main Hop Field in the corresponding AS entry. To make this work, the MAC of the peering Hop Field is also chained to the MAC of the main Hop Field - this allows to validate the chained MAC for both the peering Hop Field and the following Hop Fields, by using the same `Acc` field value.

This results in the following definition for the calculation of the MAC for a peering Hop Field.

The Control Plane Internet-Draft defines a peering Hop Field as follows:

Hop Field<sup>Peer</sup><sub>i</sub> = (ExpTime<sup>Peer</sup><sub>i</sub>, ConsIngress<sup>Peer</sup><sub>i</sub>, ConsEgress<sup>Peer</sup><sub>i</sub>, MAC<sup>Peer</sup><sub>i</sub>)

This definition, the general definition of a Hop Field MAC and the explanation above leads to the following definition of the MAC for a peering Hop Field<sup>Peer</sup><sub>i</sub>:

MAC<sup>Peer</sup><sub>i</sub> = <br> Ck<sup>Peer</sup><sub>i</sub> (`SegID` XOR MAC<sub>0</sub> \[:2] ... XOR MAC<sub>i</sub> \[:2], Timestamp, ExpTime<sup>Peer</sup><sub>i</sub>, ConsIngress<sup>Peer</sup><sub>i</sub>, ConsEgress<sup>Peer</sup><sub>i</sub>)

**Note:** The XOR-sum of the MACs in the formula of the peering Hop Field **also includes** the MAC of the main Hop Field (whereas for the calculation of the MAC for the main Hop Field itself only the XOR-sum of the *previous* MACs is used).

**Note:** The Control-Plane Internet-Draft is available here: {{I-D.scion-cp}}.


## Path Initialization and Packet Processing {#packet-verif}

As is described in [](#format), the path header of the data-plane packets only contains a sequence of Info Fields and Hop Fields without any additional data from the corresponding PCBs. Also, the SCION path does not contain any AS numbers (except for the source and destination ASes), and there is no field explicitly defining the type of each segment (up, core, or down). This chapter describes the required steps for the source endpoint and each SCION router to ensure that a data packet only traverses authorized segments. The chapter first specifies the initialization of a path at the source endpoint, followed by the steps that the SCION routers needs to perform when a data-plane packet traverses an AS on its way to the destination.


### Initialization at Source Endpoint

The source endpoint needs to initialize a path correctly for the SCION routers to be able to verify the Hop Fields in the data plane. To this end, the source endpoint MUST perform the following steps:

1. Combine the preferred end-to-end path from the path segments obtained during path lookup.
2. Extract the Info Fields and Hop Fields from the different path segments that together build the end-to-end path to the destination endpoint. Then insert the relevant information from the path segments' info and Hop Fields into the corresponding `InfoFields` and `Hopfields`, respectively, in the data packet header.
3. Each 8-byte Info Field `InfoField` in the packet header contains the updatable `Acc` field as well as a Peering flag `P` and a Construction Direction flag `C` (see also [](#inffield)). As a next step in the path initialization process, the source MUST correctly set the flags and the `Acc` field of all `InfoFields` included in the path, according to the following rules:

   **Note:** As already stated above, the type of segment is not visible directly in the forwarding path but can be inferred from flags and other information. See also [](#process-router).

   - The Construction Direction flag `C` MUST be set to "1" whenever the corresponding segment is traversed in construction direction, i.e., for down-path segments and potentially for core-segments. It MUST be set to "0" for up-path segments and "reversed" core-segments.
   - The Peering flag `P` MUST be set to "1" for up- and down-segments if, and only if, the path contains a peering Hop Field.
   - The field `Acc` field is an updatable field. It is used to compute the MAC over the current Hop Field. The value of the field `Acc` field corresponds to the value of the Accumulator Acc<sub>i</sub> for AS<sub>i</sub>.  It is initialized based on the location of the sender in relation to path construction.

   The following `InfoField` settings are possible, based on the following possible use cases:

   - **Use case 1** <br> The path segment is traversed in construction direction and includes no peering Hop Field. It starts at the *i*-th AS of the full segment discovered in beaconing. In this case:

     - The Peering flag `P` = "0"
     - The Construction Direction flag `C` = "1"
     - The value of the `Acc` = Acc<sub>i</sub>. For more details, see [](#def-acc).

   - **Use case 2** <br> The path segment is traversed in construction direction and includes a peering Hop Field (which is the first Hop Field of the segment). It starts at the *i*-th AS of the full segment discovered in beaconing. In this case:

     - The Peering flag `P` = "1"
     - The Construction Direction flag `C` = "1"
     - The value of the `Acc` = Acc<sub>i+1</sub>. For more details, see [](#def-acc).

   - **Use case 3** <br> The path segment is traversed against construction direction. The full segment has "n" hops. In this case:

     - The Peering flag `P` = "0" or "1" (depending on whether the last Hop Field in the up-segment is a peering Hop Field)
     - The Construction Direction flag `C` = "0"
     - The value of the `Acc` = Acc<sub>n-1</sub>. This is because seen from the direction of beaconing, the source endpoint is the last AS in the path segment. For more details, see [](#def-mac) and [](#def-acc).

4. Besides setting the flags and the `Acc` field, the source endpoint MUST also set the pointers in the `CurrInf` and `CurrHF` fields of the Path Meta Header `PathMetaHdr` (see [](#PathMetaHdr)). As the source endpoint builds the starting point of the forwarding, both pointers MUST be set to "0".


### Processing at Routers {#process-router}

During forwarding, each AS<sub>i</sub> verifies the path contained in the packet header with the help of the current value of the MAC in the current Hop Field, and the current value of the Accumulator in the `Acc` field of the current Info Field. Additionally, each AS has to correctly set the value of the Accumulator in the `Acc` field for the next AS to be able to verify its Hop Field. The exact operations differ based on the location of the AS on the path.

The processing of SCION packets for ASes where a peering link is crossed between path segments is special cased. A path containing a peering link contains exactly two path segments, one against construction direction (up) and one in construction direction (down). On the path segment against construction direction (up), the peering Hop Field is the last hop of the segment. In construction direction (down), the peering Hop Field is the first hop of the segment.

The following sections describe the tasks to be performed by the ingress and egress border router of each on-path AS. Each operation is described from the perspective of AS<sub>i</sub>, where i belongs to \[0 ... n-1], and n == the number of ASes in the path segment (counted from the first AS in the beaconing direction).

**Note:** In this context, a border router is called **ingress** border router when it refers to an entrance border router to an AS, as seen from the direction of travel of the SCION packet. So in the context here, the ingress border router is the *(packet) incoming* border router. A border router is called **egress** border router when it refers to an exit border router of an AS, as seen from the direction of travel of the SCION packet. So in this context, the egress border router is the *(packet) leaving* border router.

The following figure provides a simplified representation of the processing at routers both in construction direction and against construction direction.

~~~~
                              .--.
                             ( RR )  = Router
Processing in                 `--'
construction
direction

      1. Verify MAC of AS1          1. Verify MAC of AS2
      2. Update Acc for AS2         2. Update Acc for AS3
                 |                            |
>>>--------------o----------------------------o---------------------->>>

+-------------+  |           +-------------+  |          +-------------+
|             |              |             |             |             |
|           .--. |          .--.         .--. |         .--.           |
|   AS1    ( RR )o---------( RR )  AS2  ( RR )o--------( RR )  AS3     |
|           `--' |          `--'         `--' |         `--'           |
|             |              |             |             |             |
+-------------+  |           +-------------+  |          +-------------+

                 |                            |
<<<--------------o----------------------------o----------------------<<<
                 |                            |
      1. Update Acc for AS1         1. Update Acc for AS2
      2. Verify MAC of AS1          2. Verify MAC of AS2

                                                      Processing against
                                                            construction
                                                               direction
~~~~
{: #figure-19 title="A simplified representation of the processing at routers in both directions."}


#### Steps Ingress Border Router

This section describes the steps that a SCION ingress border router MUST perform when it receives a SCION packet.

1. Check that the interface through which the packet was received is equal to the ingress interface in the current Hop Field. If not, the router MUST drop the packet.
2. Check if the current Hop Field is expired or originated in the future. That is, the current Info Field MUST NOT have a timestamp in the future, as defined in [](#inffield). If either is true, the router MUST drop the packet.
3. The next steps depend on the direction of travel and whether this segment includes a peering Hop Field. Both features are indicated by the settings of the Construction Direction flag `C` and the Peering flag `P` in the current Info Field. Therefore, check the settings of both flags. The following combinations are possible:

   - The packet traverses the path segment in **construction direction** (`C` = "1" and `P` = "0" or "1"). In this case, proceed with step 4.

   - The packet traverses the path segment **against construction direction** (`C` = "0"). The following use cases are possible:

     - **Use case 1** <br> The path segment includes **no peering Hop Field** (`P` = "0"). In this case, the ingress border router MUST take the following step(s):

       - Compute the value of the Accumulator Acc as follows:

         Acc = Acc<sub>i+1</sub> XOR MAC<sub>i</sub> <br>
         where <br>
         Acc<sub>i+1</sub> = the current value of the field `Acc` in the current Info Field <br>
         MAC<sub>i</sub> = the value of MAC<sub>i</sub> in the current Hop Field representing AS<sub>i</sub>

         **Note:** In the case described here the packet travels against direction of beaconing. That is, the packet comes from AS<sub>i+1</sub> and is going to enter AS<sub>i</sub>. This means that the `Acc` field of this incoming packet represents the value of Accumulator Acc<sub>i+1</sub>. However, to compute the MAC<sub>i</sub> for the current AS<sub>i</sub>, we need the value of Accumulator Acc<sub>i</sub> (see [](#def-acc)). Because the border router knows that the formula for Acc<sub>i+1</sub> = Acc<sub>i</sub> XOR MAC<sub>i</sub> \[:2] (see also [](#def-acc)), and because the values of Acc<sub>i+1</sub> and MAC<sub>i</sub> are known, the router will be able to recover the value Acc<sub>i</sub> based on the just-mentioned formula for Acc.

       - Replace the current value of the field `Acc` in the current Info Field with the newly calculated value of Acc.
       - Compute the MAC<sup>Verify</sup><sub>i</sub> over the Hop Field of the current AS<sub>i</sub>. For this, use the formula in [](#def-mac), but replace `SegID XOR MAC_0[:2] ... XOR MAC_i-1 [:2]` in the formula with the value of the accumulator Acc as just set in the `Acc` field in the current Info Field.
       - Check that the MAC<sub>i</sub> in the current Hop Field matches the just-calculated MAC<sup>Verify</sup><sub>i</sub>. If yes, it is fine. Otherwise, drop the packet.
       - Check whether the current Hop Field is the last Hop Field in the path segment. For this, look at the value of the current `SegLen` and other metadata in the path meta header. If yes, increment both `CurrInf` and `CurrHF` in the path meta header. Proceed with step 4.

     - **Use case 2** <br> The path segment includes a **peering Hop Field** (`P` = "1"), but the current hop is **not** the peering hop, that is, the current Hop Field is **not** the *last* Hop Field of the segment, seen from the direction of travel - this can be determined by looking at the value of the current `SegLen` and other metadata in the path meta header. In this case, the ingress border router needs to perform the steps previously described for the path segment without peering Hop Field. However, the border router MUST NOT increment `CurrInf` and MUST NOT increment `CurrHF` in the path meta header. Proceed with step 4.

     - **Use case 3** <br> The path segment includes a **peering Hop Field** (`P` = "1"), and the current Hop Field *is* the peering Hop Field. This would be the case if the current Hop Field is the *last* Hop Field of the segment, seen from the direction of travel - to find out whether this is true, check the value of the current `SegLen` and other metadata in the path meta header. In this case, the ingress border router MUST take the following step(s):

       - Compute MAC<sup>Peer</sup><sub>i</sub>. For this, use the formula in [](#peerlink), but replace `SegID XOR MAC_0[:2] ... XOR MAC_i [:2]` in the formula with the value of the accumulator Acc as set in the `Acc` field in the current Info Field (this is the value of the accumulator Acc as it comes with the packet).
       - Check that the MAC<sub>i</sub> in the current Hop Field matches the just-calculated MAC<sup>Peer</sup><sub>i</sub>. If yes, it is fine. Otherwise, drop the packet.
       - Increment both `CurrInf` and `CurrHF` in the path meta header. Proceed with step 4.

4. Forward the packet to the egress border router (based on the egress interface ID in the current Hop Field) or to the destination endpoint, if this is the destination AS.

**Note:** For more information on the path meta header, see [](#PathMetaHdr).


#### Steps Egress Border Router

This section describes the steps that a SCION egress border router MUST perform when it receives a SCION packet.

1. Parse the SCION packet.
2. The next steps depend on the direction of travel and whether this segment includes a peering link. Both features are indicated by the settings of the Construction Direction flag `C` and the Peering flag `P` in the currently valid Info Field. Therefore, first check the settings of both flags. The following use cases are possible:

   - **Use case 1** <br> The packet traverses the path segment in **construction direction** (`C` = "1"). The path segment either includes **no peering Hop Field** (`P` = "0"), or the path segment does include a **peering Hop Field** (`P` = "1"), but the current hop is **not** the peering hop, that is, the current Hop Field is **not** the *first* Hop Field of the segment, seen from the direction of travel. To check whether this is true, look at the value of the current `SegLen` and other metadata in the path meta header. In this case, the egress border router MUST take the following step(s):

     - Compute MAC<sup>Verify</sup><sub>i</sub> over the Hop Field of the current AS<sub>i</sub>. For this, use the formula in [](#def-mac), but replace `SegID XOR MAC_0[:2] ... XOR MAC_i-1 [:2]` in the formula with the value of the accumulator Acc as set in the `Acc` field in the current Info Field.
     - Check that the just-calculated MAC<sup>Verify</sup><sub>i</sub> matches MAC<sub>i</sub> in the Hop Field of the current AS<sub>i</sub>. If yes, it is fine. Otherwise, drop the packet.
     - Compute the value of Acc<sub>i+1</sub>. For this, use the formula in [](#def-acc). Replace Acc<sub>i</sub> in the formula with the current value of the accumulator Acc as set in the `Acc` field of the current Info Field.
     - Replace the value of the `Acc` field in the current Info Field with the just-calculated value of Acc<sub>i+1</sub>.
     - Proceed with step 3.

   - **Use case 2** <br> The packet traverses the path segment in **construction direction** (`C` = "1"). The path segment includes a **peering Hop Field** (`P` = "1"), and the current Hop Field *is* the peering Hop Field. This would be the case if the current Hop Field is the *first* Hop Field of the segment, seen from the direction of travel - to find out whether this is true, check the value of the current `SegLen` and other metadata in the path meta header. In this case, the egress border router MUST take the following steps:

     - Compute MAC<sup>Peer</sup><sub>i</sub>. For this, use the formula in [](#peerlink), but replace `SegID XOR MAC_0 [:2] ... XOR MAC_i [:2]` with the value in the `Acc` field of the current Info Field.
     - Check that the MAC<sub>i</sub> in the Hop Field of the current AS<sub>i</sub> matches the just-calculated MAC<sup>Peer</sup><sub>i</sub>. If yes, it is fine - proceed with step 3. Otherwise, drop the packet.

   - **Use case 3** <br> The packet traverses the path segment **against construction direction** (`C` = "0" and `P` = "0" or "1"). In this case, proceed with the next step, step 3.

3. Increment `CurrHF` in the path meta header.
4. Forward the packet to the neighbor AS.

**Note:** For more information on the path meta header, see [](#PathMetaHdr).

#### Effects of Clock Inaccuracy

A PCB originated by a given control service is used to construct data plane paths. Specifically, the timestamp in the Info Field and the expiry time of Hop Fields are used for Hop Field MAC computation, see [](#hf-mac-calc), which is used to validate paths at each on-path SCION router. A segment's originating control service and the routers that the segment refers to all have different clocks. Their differences affect the validation process:

* A fast clock at origination or a slow clock at validation will yield a lengthened expiration time for hops, and possibly an origination time in the future.
* A slow clock at origination or a fast clock at validation will yield a shortened expiration time for hops, and possibly an expiration time in the past.

This bias comes in addition to a structural delay: PCBs are propagated at a configurable interval (typically, one minute). As a result of this and the way they are iteratively constructed, PCBs with N hops may become available for path construction up to N intervals (so typically N minutes) after origination. This creates a constraint on the expiration of hops. Hops of the minimal expiration time (337.5 seconds - see [](#hopfld)) would render useless any path segment longer than 5 hops. For this reason, it is unadvisable to create hops with a short expiration time. The norm is 6 hours.

In comparison to these time scales, clock offsets in the order of minutes are immaterial.

Each administrator of SCION control services and routers is responsible for maintaining sufficient clock accuracy. No particular method is assumed by this specification.

# SCMP {#scmp}

## Introduction

The SCION Control Message Protocol (SCMP) is analogous to the Internet Control Message Protocol (ICMP). It provides functionality for network diagnostics, such as traceroute, and error messages that signal packet processing or network-layer problems. SCMP is a helpful tool for network diagnostics and, in the case of External Interface Down and Internal Connectivity Down messages, an optimization for end hosts to detect network failures more rapidly and fail-over to different paths. However, SCION nodes should not strictly rely on the availability of SCMP, as this protocol may not be supported by all devices and/or may be subject to rate limiting.

This document specifies only messages RECOMMENDED for the purposes of path diagnosis and recovery. An extended specification, still a work in progress, can be found in {{SCMP}}.

## General Format

Every SCMP message is preceded by a SCION header, and zero or more SCION extension headers. The SCMP header is identified by a `NextHdr` value of `202` in the immediately preceding header.

The messages have the following general format:

~~~~

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     Type      |     Code      |           Checksum            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                       Type-dependent Block                    |
    +                                                               +
    |                         (variable length)                     |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~
{: #figure-21 title="SCMP message format"}

*Type* indicates the type of SCMP message. Its value determines the format of the info and data block.

*Code* provides additional granularity to the SCMP type.

*Checksum* is used to detect data corruption.

*InfoBlock* is an optional field of variable length. The format is dependent on the message type.

*DataBlock* is an optional field of variable length. The format is dependent on the message type.

## Message Types

SCMP messages are grouped into two classes: error messages and informational messages. Error messages are identified by a zero in the high-order bit of the type value. I.e., error messages have a type value in the range of 0-127. Informational messages have type values in the range of 128-255.

This specification defines the message formats for the following SCMP messages:

**SCMP error messages**:

|Type | Meaning                                                   |
|-----+-----------------------------------------------------------|
|1    | Reserved for future use                                   |
|2    | [Packet Too Big](#packet-too-big)                         |
|3    | Reserved for future use                                   |
|4    | Reserved for future use                                   |
|5    | [External Interface Down](#external-interface-down)       |
|6    | [Internal Connectivity Down](#internal-connectivity-down) |
|     |                                                           |
|100  | Private Experimentation                                   |
|101  | Private Experimentation                                   |
|     |                                                           |
|127  | Reserved for expansion of SCMP error messages             |
{: title="type values"}

**SCMP informational messages**:

| Type | Meaning                                                  |
|------+----------------------------------------------------------|
| 128  | Reserved for future use                                  |
| 129  | Reserved for future use                                  |
| 130  | [Traceroute Request](#traceroute-request)                |
| 131  | [Traceroute Reply](#traceroute-reply)                    |
| 200  | Private Experimentation                                  |
| 201  | Private Experimentation                                  |
|      |                                                          |
| 255  | Reserved for expansion of SCMP informational messages    |
{: title="type values"}

Type values 100, 101, 200, and 201 are reserved for private experimentation.
They are not intended for general use. Any wide-scale and/or uncontrolled usage
should obtain a real allocation.

Type values 127 and 255 are reserved for future expansion of in case of a
shortage of type values.

## Checksum Calculation

The checksum is the 16-bit one's complement of the one's complement sum of the
entire SCMP message, starting with the SCMP message type field, and prepended
with a "pseudo-header" consisting of the SCION address header and the layer-4
protocol type as defined in [](#pseudo).

## Processing Rules

Implementations MUST respect the following rules when processing SCMP messages:

   - If an SCMP error message of unknown type is received at its destination, it MUST be passed to the upper-layer process that originated the packet that caused the error, if it can be identified.
   - If an SCMP informational message of unknown type is received, it MUST be silently dropped.
   - Every SCMP error message MUST include as much of the offending SCION packet as possible without making the error message packet - including the SCION header and all extension headers - exceed **1232 bytes**.
   - In case the implementation is required to pass an SCMP error message to the upper-layer process, the upper-layer protocol type is extracted from the original packet in the body of the SCMP error message and used to select the appropriate process to handle the error. In case the upper-layer protocol type cannot be extracted from the SCMP error message body, the SCMP message MUST be silently dropped.
   - An SCMP error message MUST NOT be originated in response to any of the following:
     - An SCMP error message.
     - A packet which source address does not uniquely identify a single node. E.g., an IPv4 or IPv6 multicast address.

## Error Messages {#scmp-notification}

### Packet Too Big {#packet-too-big}

~~~~

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     Type      |     Code      |          Checksum             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |            reserved           |             MTU               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                As much of the offending packet                |
    +              as possible without the SCMP packet              +
    |                    exceeding 1232 bytes.                      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~
{:figure-24 title="packet-too-big}

| SCMP Fields  |                                                     |
|--------------+-----------------------------------------------------|
| Type         | 2                                                   |
| Code         | 0                                                   |
| MTU          | The Maximum Transmission Unit of the next-hop link. |
{: title="field values"}

A **Packet Too Big** message SHOULD be originated by a router in response to a
packet that cannot be forwarded because the packet is larger than the MTU of the
outgoing link. The MTU value is set to the maximum size a SCION packet can have
to still fit on the next-hop link, as the sender has no knowledge of the
underlay.

### External Interface Down {#external-interface-down}

~~~~

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     Type      |     Code      |          Checksum             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |              ISD              |                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         AS                    +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    +                        Interface ID                           +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                As much of the offending packet                |
    +              as possible without the SCMP packet              +
    |                    exceeding 1232 bytes.                      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~
{: #figure-22 title="External-interface-down-format"}

| SCMP Fields  |                                                               |
|--------------+---------------------------------------------------------------|
| Type         | 5                                                             |
| Code         | 0                                                             |
| ISD          | The 16-bit ISD identifier of the SCMP originator              |
| AS           | The 48-bit AS identifier of the SCMP originator               |
| Interface ID | The interface ID of the external link with connectivity issue.|
{: title="field values"}

A **External Interface Down** message SHOULD be originated by a router in response
to a packet that cannot be forwarded because the link to an external AS is broken.
The ISD and AS identifier are set to the ISD-AS of the originating router.
The interface ID identifies the link of the originating AS that is down.

Recipients can use this information to route around broken data-plane links.

### Internal Connectivity Down {#internal-connectivity-down}

~~~~

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     Type      |     Code      |          Checksum             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |              ISD              |                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         AS                    +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    +                   Ingress Interface ID                        +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    +                   Egress Interface ID                         +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                As much of the offending packet                |
    +              as possible without the SCMP packet              +
    |                    exceeding 1232 bytes.                      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~
{: #figure-23 title="internal-connectivity-down-format"}

| SCMP Fields  |                                                               |
|--------------+---------------------------------------------------------------|
| Type         | 6                                                             |
| Code         | 0                                                             |
| ISD          | The 16-bit ISD identifier of the SCMP originator              |
| AS           | The 48-bit AS identifier of the SCMP originator               |
| Ingress ID   | The interface ID of the ingress link.                         |
| Egress ID    | The interface ID of the egress link.                          |
{: title="field values"}

A **Internal Connectivity Down** message SHOULD be originated by a router in
response to a packet that cannot be forwarded inside the AS because because the
connectivity between the ingress and egress routers is broken. The ISD and AS
identifier are set to the ISD-AS of the originating router. The ingress
interface ID identifies the interface on which the packet enters the AS. The
egress interface ID identifies the interface on which the packet is destined to
leave the AS, but the connection is broken to.

Recipients can use this information to route around broken data-plane inside an
AS.

## Informational Messages {#scmp-information}

### Traceroute Request {#traceroute-request}

~~~~

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     Type      |     Code      |          Checksum             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           Identifier          |        Sequence Number        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |              ISD              |                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         AS                    +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    +                          Interface ID                         +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~
{: #figure-24 title="traceroute-request-format"}

| SCMP Fields  |                                                               |
|--------------+---------------------------------------------------------------|
| Type         | 130                                                           |
| Code         | 0                                                             |
| Identifier   | A 16-bit identifier to aid matching replies with requests     |
| Sequence Nr. | A 16-bit sequence number to aid matching replies with request |
| ISD          | Place holder set to zero by SCMP sender                       |
| AS           | Place holder set to zero by SCMP sender                       |
| Interface ID | Place holder set to zero by SCMP sender                       |
{: title="field values"}

A border router is alerted of a Traceroute Request message through the ConsIngress or ConsEgress Router Alert flag in the hop field that describes the traversal of that router in a packet's path. Senders have to set these flags appropriately. When such a packet is received, the border router SHOULD reply with a [Traceroute Reply message](#traceroute-reply).

### Traceroute Reply {#traceroute-reply}

~~~~

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     Type      |     Code      |          Checksum             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           Identifier          |        Sequence Number        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |              ISD              |                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         AS                    +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    +                          Interface ID                         +
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~
{: #figure-25 title="traceroute-reply-format"}

| SCMP Fields                                                                  |
|--------------+---------------------------------------------------------------|
| Type         | 131                                                           |
| Code         | 0                                                             |
| Identifier   | The identifier set in the Traceroute Request                  |
| Sequence Nr. | The sequence number of the Tracroute Request                  |
| ISD          | The 16-bit ISD identifier of the SCMP originator              |
| AS           | The 48-bit AS identifier of the SCMP originator               |
| Interface ID | The interface ID of the SCMP originating router               |
{: title="field values"}

The identifier is set to the identifier value from the [Traceroute Request message](#traceroute-request). The ISD and AS identifiers are set to the ISD-AS of the originating border router.

## Authentication

Authentication of SCMP packets is not specified. In a context where endpoints may depend on SCMP messages (at least [External Interface Down](#external-interface-down) and [Internal Connectivity Down](#internal-connectivity-down)) for reliable operations, this is recognized as a problem and is being addressed.

# Failure To Forward

## Link Failure Detection - BFD {#scion-bfd}

One of the reasons not to forward a packet is the failure of the link through which the packet needs to egress. Whether such failures are detectable upon sending depends on the underlay protocol being used. In the case of UDP/IP, it is common for failures to go undetected. To detect link failures more reliably, SCION uses the Bidirectional Forwarding Detection (BFD) protocol ({{RFC5880}} and {{RFC5881}}). BFD is a protocol intended to detect faults in the bidirectional path between two forwarding engines, with typically very low latency. It operates independently of media, data protocols, and routing protocols.

A SCION router monitor the links at each of its external interfaces by exchanging BFD messages with its peer. A SCION router monitors its connectivity with other routers in the same AS by exchanging BFD messages with them. A SCION BFD message is a SCION packet with a `NextHdr` value of `203` (`BFD/SCION`) and a path type of either `00` (`Empty` - used on internal links) or `2` (`OneHopPath` - used on inter-as links). The BFD header itself is a BFD Control Header as described in {{RFC5880}}.

More information is available on one-hop and empty paths in [](#onehop) and [](#empty).

The protocol between peers is as descibed in {{RFC5880}}. To summarize: if a node does not receive a BFD message from its peer at some regular interval, it considers the link to be down (in both directions) until messages are received again.

A SCION router should accept BFD connections from its peers and SHOULD establish BFD connections to its peers. While a link is considered to be down, a SCION router should drop packets that must egress via that link. In that case, it SHOULD send a [notification](#link-down-notification) to the originator.

## Notification - SCMP {#link-down-notification}

If a failure to forward a packet is the result of a local condition, the path used by the packet is unusable. As a result, the originator cannot communicate with the recipient until the fault is repaired, the path expires, or a different path is used. SCION is designed so that a router cannot change the path followed by a packet. Only the originating endpoint can chose a different path. Therefore, to enable fast recovery, a router SHOULD notify the originator, via an [SCMP notifications](#scmp-notification), if the packet is dropped due to a local, non-transient, fault.

Sending an SCMP error notification is never mandatory. To reduce exposure to denial-of-service attacks, a SCION router SHOULD restrict the amount of work done when dropping a packet. In most cases the packet SHOULD be dropped without performing any additional work. Rate-limiting the preparation and sending of recommended SCMP notifications (especially identical ones) is good practice. Rate limit policies are up to each AS' administrator.

# Security Considerations

This section describes the possible security risks and attacks that SCION's data plane may be prone to, and how these attacks may be mitigated. It first discusses security risks that pertain to path authorization, followed by a section on other forwarding-related security considerations.

## Path Authorization

A central property of the SCION path-aware data plane is path authorization. Path authorization guarantees that data packets always traverse the network along path segments authorized in the control plane by all on-path ASes. This section discusses how an adversary may attempt to violate the path-authorization property, as well as SCION's prevention mechanisms to these attacks; either an attacker can attempt to create unauthorized Hop Fields, or they can attempt to create illegitimate paths assembled from authentic individual Hop Fields.

The main protection mechanism here is the Hop Field MAC (see [](#auth-chained-macs)), authenticating the Hop Field content, consisting of ingress/egress interface identifiers, creation and expiration timestamp and, virtually, the preceding Hop Field MACs in the path segment.
Recall that each Hop Field MAC is computed using the respective AS's secret forwarding key, which is shared across the SCION routers and control plane services within each AS.

### Forwarding key compromise

For the current default MAC algorithm, AES-CMAC truncated to 48 bits, key recovery attacks from (any number of) known plaintext/MAC combinations is computationally infeasible, as far as publicly known.
In addition, the MAC algorithm can be freely chosen by each AS, enabling algorithmic agility for MAC computations. Should a MAC algorithm be discovered to be weak or insecure, each AS can quickly switch to a secure algorithm without the need for coordination with other ASes.

A more realistic risk to the secrecy of the forwarding key is exfiltration from a compromised router or control plane service.
An AS can optionally rotate its forwarding key at regular intervals to limit the exposure after a temporary device compromise. However, as is perhaps self-evident, such a key rotation scheme cannot mitigate the impact of an undiscovered, permanent compromise of a device.

When an AS's forwarding key is compromised, an attacker can forge Hop Field MACs, undermining path authorization. Recall that path segments are checked for validity and policy compliance during the path discovery phase, and during forwarding, routers only validate the MAC and basic validity of the current the Hop Field. Consequently, creating fraudulent Hop Fields with valid MACs allows an attacker to bypass most path segment validity checks, and to create path segments that violate the AS's local policy and/or general path segment validity requirements.
In particular, an attacker could create paths that include loops (limited by the maximum number of Hop Fields of a path).

Unless an attacker has access to the forwarding keys of all ASes on the illegitimate path it wants to fabricate, it will need to splice fragments of two legitimate path segments with an illegitimate Hop Field. For this, it needs to create a Hop Field with a MAC that fits into the MAC chain expected by the second path segment fragment. The only input that the attacker can vary relatively freely is the 8 bit ``ExpTime``, but the resulting MAC needs to match a specific 16 bit ``Acc`` value. While there is a low probability of this working for a specific attempt (1/256), the attack will succeed eventually if the attacker can keep retrying over a longer time period or with many different path segment fragments.

While a forwarding key compromise and the resulting loss of path authorization is a serious degradation of SCION's routing security properties, this does not affect, for example, access control or data security for the hosts in the affected AS.
Unauthorized paths are available to the attacker, but the routing of packets from legitimate senders is not affected.

### Forging Hop Field MAC

As a second method to break path authorization is to directly forge a Hop Field in an online attack, using the router as an oracle to determine the validity of the Hop Field MAC. The adversary needs to send one packet per guess for verification. For a 6-byte MAC, the adversary would need an expected 2<sup>47</sup> (~140 trillion) tries to successfully forge the MAC of a single Hop Field.
As the router only checks MACs during the encoded validity period of the Hop Field, which is limited by the packet header format to at most 24 hours, these tries need to occur in a limited time period. This results in a seemingly infeasible number of ~1.6e9 guesses per second.
In the unlikely case that an online brute-force attack succeeds, the obtained Hop Field can be used until its inevitable expiration after the just mentioned 24 hour limit.

### Path Splicing {#path-splicing}

In a path-splicing attack, an adversary source endpoint takes valid Hop Fields of multiple path segments and splices them together to obtain a new unauthorized path.

Two candidates path segments for splicing must have at least one AS interface in common as a connection point.
The path segments must have the same origination timestamp, as this is directly protected by the Hop Field MAC. This can occur by chance, or if the two candidate path segments were originated as the same segment that then diverged and converged back.
Finally, the Hop Field MAC protects the 16-bit aggregation of path segment identifier and preceding MACs. For details, see [](#auth-chained-macs). This MAC chaining prevents splicing even in the case that the AS interface and segment timestamp match.

As the segment identifier and aggregation of preceding MACs is only 16-bits wide, per-chance collision among compatible path segments can occur.
With typical network sizes and numbers of paths of today, such collisions might occur rarely.
Successful path splicing would allow an attacker to briefly use a path that violates an ASes path policy, e.g. making a special transit link available to a customer AS that is not billed accordingly, or violate general path segment validity requirements. In particular, a spliced path segment could traverse one or multiple links twice. However, creating a loop traversing a link an arbitrary number of times would involve multiple path splices and therefore multiple random collisions happening simultaneously, which is exceedingly unlikely.
A wider security margin against path splicing could be obtained by increasing the width of the segment identifier / `Acc` field, e.g. by extending it into the 8-bit reserved field next to it in the Info Field.


## On-Path Attacks

When an adversary sits on the path between the source and destination endpoint, it is able to intercept the data packets that are being forwarded. This would allow the adversary to hijack traffic onto a path that is different from the intended one selected by the source endpoint. Possible on-path attacks in the data plane are modifications of the SCION path header and SCION address header. In addition, an on-path adversary can always simply drop packets. This kind of attack is fundamental and generally cannot be prevented. However, in this case, the endpoint can use SCION's path-awareness to immediately select an alternate path if available.


### Modification of the Path Header

An on-path adversary could modify the SCION path header, and replace the remaining part of path segments to the destination with different segments. Such replaced segments must include authorized segments, as otherwise the packet would be simply dropped on its way to the destination. The already traversed portion of the current segment and past segments can also be modified by the adversary (for instance, deleting and adding valid and invalid Hop Fields). On reply packets from the destination, the adversary can transparently revert the changes to the path header again. For instance, if an adversary M is an intermediate AS on the path of a packet from A to B, then M can replace the packet’s past path (leading up to, but not including M). The new path may not be a valid end-to-end path. However, when B reverses the path and sends a reply packet, that packet would go via M, which can then transparently change the invalid path back to the valid path to A. In addition, the endpoint address header can also be modified.

Modifications of the SCION path and address header can be discovered by the destination endpoint by a data integrity protection system. Such a data integrity protection system, loosely analogous to the IPSec Authentication Header, exists for SCION but is out of scope for this document. This is described as the SCION Packet Authentication Option (SPAO) in [CHUAT22].


Moreover, packet integrity protection is not enough if there are two colluding adversaries on the path. These colluding adversaries can forward the packet between them using a different path than selected by the source endpoint: The first on-path attacker remodels the packet header arbitrarily, and the second on-path attacker changes the path back to the original source-selected path, such that the integrity check by the destination endpoint succeeds. However, such an attack is of little value. An on-path adversary may inspect/copy/disrupt its traffic without diverting it away from the sender-chosen path. For this reason proof-of-transit, which would be required to detect such an attack, has marginal benefit in the context of SCION and it is not in scope for this document.


## Off-Path Attacks

SCION's path-awareness limits the abilities of an off-path adversary to influence forwarding in the data plane. Once a packet is "in flight", it will follow its set route, no matter what an adversary may do.
An adversary can attempt to disrupt the connectivity of said path by flooding a link with excessive traffic (see [](#dos) below). After detecting congestion, the endpoint can switch to another, non-congested path for subsequent packets.


## Volumetric Denial of Service Attacks {#dos}

An adversary can attempt to disrupt the connectivity of a network path by flooding a link with excessive traffic. In this case, the endpoint can switch to another, non-congested path for subsequent packets.

SCION provides protection against certain reflection-based DoS attacks. Here, the adversary sends requests to a server with the source address set to the address of the victim. The server will send a reply, typically larger than the request, to the victim. As long as the attacker and the victim are located in different ASes, this can be prevented in SCION. The reply packets are simply returned along reversed path to the actual sender, regardless of the source address information. Thus, the reflected will be forwarded to the attackers AS (where it will be discarded because the destination AS does not match).

On the flip side, the path choice of the endpoint may possibly be exploited by an attacker to create intermittent congestion with relatively low send rate; the attacker can abuse the latency differences of the available paths, sending at precisely timed intervals to cause short, synchronized bursts of packets near the victim.

**Note** that SCION does not protect against two other types of DoS attacks, namely transport protocol attacks and application layer attacks. Such attacks are out of SCION's scope. However, the additional information contained in the SCION header enables more targeted filtering, e.g., by ISD, AS or path length.


# Interoperability: SCION IP Gateway {#sig}

The SCION IP Gateway (SIG) enables IP packets to be tunnelled over SCION to support the use of applications that are not SCION enabled.

An ingress SIG encapsulates IP packets within SCION packets and sends them across a SCION network to an egress SIG. The egress SIG decapsulates the IP packets from the SCION packets and forwards them towards their destination IP address. The SIGs at either end of a tunnel act as routers from the perspective of IP, whilst acting as SCION endpoints from the perspective of the SCION network.

Each SIG establishes a session to one or multiple remote SIGs. Each pair of SIGs uses a common tunneling protocol. Each SIG may choose to send SCION packets to a remote SIG in accordance with static IP routes, or by dynamically announcing IP prefixes to each other via a routing protocol. Each SIG may also choose how to send SCION packets based on locally configured policies when multiple remote SIGs are available.
In addition, the source SIG is responsible for SCION path selection.

A SIG is typically deployed inside the same AS internal network as its non-SCION hosts. In an enterprise scenario, it is usually deployed at the edge of the enterprise network.

# IANA Considerations

This document has no IANA actions.

The SCION AS and ISD number are SCION-specific numbers. They are currently allocated by Anapaya Systems, a provider of SCION-based networking software and solutions (see [Anapaya ISD AS assignments](https://docs.anapaya.net/en/latest/resources/isd-as-assignments/)). This task is currently being transitioned from Anapaya to the SCION Association.


--- back

# Acknowledgments
{:numbered="false"}

Many thanks go to Matthias Frei (SCION Association), Juan A. Garcia Prado (ETH Zurich), Kevin Meynell (SCION Association) and Jean-Christophe Hugly (SCION Association) for reviewing this document. We are also very grateful to Adrian Perrig (ETH Zurich), for providing guidance and feedback about each aspect of SCION. Finally, we are indebted to the SCION development teams of Anapaya and ETH Zurich, for their practical knowledge and for the documentation about the SCION Data Plane, as well as to the authors of [CHUAT22] - the book is an important source of input and inspiration for this draft.


# Assigned SCION Protocol Numbers {#protnum}
{:numbered="false"}

This appendix lists the assigned SCION protocol numbers.


## Considerations
{:numbered="false"}

SCION attempts to take the IANA's assigned Internet protocol numbers into consideration. Widely employed protocols have the same protocol number as the one assigned by IANA. SCION specific protocol numbers start at 200.

The protocol numbers are used in the SCION header to identify the upper layer protocol.

## Assignment
{:numbered="false"}


| Decimal   | Keyword      | Protocol                                 |
|-----------+--------------+------------------------------------------|
| 0-5       |              | Unassigned                               |
| 6         | TCP/SCION    | Transmission Control Protocol over SCION |
| 7-16      |              | Unassigned                               |
| 17        | UDP/SCION    | User Datagram Protocol over SCION        |
| 18-199    |              | Unassigned                               |
| 200       | HBH          | SCION Hop-by-Hop Options                 |
| 201       | E2E          | SCION End-to-End Options                 |
| 202       | SCMP         | SCION Control Message Protocol           |
| 203       | BFD/SCION    | BFD over SCION                           |
| 204-252   |              | Unassigned                               |
| 253       |              | Use for experimentation and testing      |
| 254       |              | Use for experimentation and testing      |
| 255       |              | Reserved                                 |
{: title="The assigned SCION protocol numbers"}
