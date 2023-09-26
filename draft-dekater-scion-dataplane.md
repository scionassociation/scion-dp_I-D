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
     email: cdk@scion.org

 -   ins: M. Frei
     name: Matthias Frei
     org: SCION Association
     email: matzf@scion.org

 -   ins: N. Rustignoli
     name: Nicola Rustignoli
     org: SCION Association
     email: nic@scion.org


normative:
  RFC1122:
  RFC1918:
  RFC2119:
  RFC2474:
  RFC2711:
  RFC3168:
  RFC4493:
  RFC5880:
  RFC5881:
  RFC8174:
  RFC8200:
  RFC9473:
  I-D.scion-cp:
    title: SCION Control Plane
    date: 2023
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
        ins: M. Frei
        name: Matthias Frei
        org: SCION Association

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
  I-D.scion-components:
    title: SCION Components Analysis
    date: 2023
    target: https://datatracker.ietf.org/doc/draft-rustignoli-panrg-scion-components/
    author:
      -
        ins: N. Rustignoli
        name: Nicola Rustignoli
        org: SCION Association
      -
        ins: C. de Kater
        name: Corine de Kater
        org: SCION Association
  I-D.scion-cppki:
    title: SCION Control-Plane PKI
    date: 2023
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
  I-D.scion-overview:
    title: SCION Overview
    date: 2023
    target: https://datatracker.ietf.org/doc/draft-dekater-panrg-scion-overview/
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
        ins: A. Perrig
        name: Adrian Perrig
        org: ETH Zuerich


--- abstract

This document describes the data plane of the path-aware, inter-domain network architecture SCION (Scalability, Control, and Isolation On Next-generation networks). One of the basic characteristics of SCION is that it gives path control to endpoints. The SCION control plane is responsible for discovering these paths and making them available as path segments to the endpoints. The responsibility of the **SCION data plane** is to combine the path segments into end-to-end paths, and forward data between endpoints according to the specified path.

The SCION data plane fundamentally differs from today’s IP-based data plane in that it is *path-aware*: In SCION, interdomain forwarding directives are embedded in the packet header. This document first provides a detailed specification of the SCION data packet format as well as the structure of the SCION header. SCION also supports extension headers - these are described, too. The document then shows the life cycle of a SCION packet while traversing the SCION Internet. This is followed by a specification of the SCION path authorization mechanisms and the packet processing at routers. SCION also includes its own protocol to communicate failures to endpoints, the SCION Control Message Protocol (SCMP). This protocol will be described in a separate document, which will follow soon.


--- middle

# Introduction

SCION leverages source-based path selection, where path information is embedded in the packet header - packet-carried forwarding state (PCFS). This section explains how endpoints can construct end-to-end paths from path segments, how the SCION inter-domain routing differs from intra-domain routing, and how data packets are forwarded through the network. It also briefly touches the concept of path authorization, which ensures that data packets always traverse the network along authorized paths.

**Note:** This is the very first version of the SCION Data Plane document. We are aware that the draft is far from perfect, and hope to improve the content in later versions of the draft. To reach this goal, any feedback is welcome and much appreciated. Thanks!

**Note:** It is assumed that readers of this draft are familiar with the basic concepts of the SCION next-generation inter-domain network architecture. If not, please find more detailed information in the IETF Internet Drafts {{I-D.scion-overview}}, {{I-D.scion-components}}, {{I-D.scion-cppki}}, and {{I-D.scion-cp}}, as well as in {{CHUAT22}}, especially Chapter 2. A short description of the SCION basic terms and elements can be found in [](#terms) below.


## Terminology {#terms}

**Autonomous System (AS)**: An autonomous system is a network under a common administrative control. For example, the network of an Internet service provider, company, or university can constitute an AS.

**Core AS**: Each isolation domain (ISD) is administered by a set of distinguished autonomous systems (ASes) called core ASes, which are responsible for initiating the path-discovery and -construction process (in SCION called "beaconing").

**Data Plane**: The data plane (sometimes also referred to as the forwarding plane) is responsible for forwarding data packets that endpoints have injected into the network. After routing information has been disseminated by the control plane, packets are forwarded according to such information by the data plane.

**Endpoint**: An endpoint is the start- or the endpoint of a SCION path. For example, an endpoint can be a host as defined in {{RFC1122}}, or a gateway bridging a SCION and an IP domain. This definition is based on the definition in {{RFC9473}}.

**Forwarding Path**: A forwarding path is a complete end-to-end path between two SCION hosts, which is used to transmit packets in the data plane and can be created with a combination of up to three path segments (an up-segment, a core-segment, and a down-segment).

**Forwarding Key**: A forwarding key is a symmetric key that is shared between the control service (control plane) and the routers (data plane) of an AS. It is used to authenticate hop fields in the end-to-end SCION path. The forwarding key is an AS-local secret and is not shared with other ASes.

**Hop Field (HF)**: As they traverse the network, path-segment construction beacons (PCBs) accumulate cryptographically protected AS-level path information in the form of hop fields. In the data plane, hop fields are used for packet forwarding: they contain the incoming and outgoing interface IDs of the ASes on the forwarding path.

**Info Field (INF)**: Each path-segment construction beacon (PCB) contains a single info field, which provides basic information about the PCB. Together with hop fields (HFs), info fields are used to create forwarding paths.

**Isolation Domain (ISD)**: In SCION, autonomous systems (ASes) are organized into logical groups called isolation domains or ISDs. Each ISD consists of ASes that span an area with a uniform trust environment (e.g., a common jurisdiction). A possible model is for ISDs to be formed along national boundaries or federations of nations.

**Leaf AS**: An AS at the "edge" of an ISD, with no other downstream ASes.

**Path Authorization**: A requirement for the data plane is that endpoints can only use paths that were constructed and authorized by ASes in the control plane. This property is called path authorization. The goal of path authorization is to prevent endpoints from crafting hop fields (HFs) themselves, modifying HFs in authorized path segments, or combining HFs of different path segments.

**Path Control**: Path control is a property of a network architecture that gives endpoints the ability to select how their packets travel through the network. Path control is stronger than path transparency.

**Path Segment**: Path segments are derived from path-segment construction beacons (PCBs). A path segment can be (1) an up-segment (i.e., a path between a non-core AS and a core AS in the same ISD), (2) a down-segment (i.e., the same as an up-segment, but in the opposite direction), or (3) a core-segment (i.e., a path between core ASes). Up to three path segments can be used to create a forwarding path.

**Path-Segment Construction Beacon (PCB)**: Core ASes generate PCBs to explore paths within their isolation domain (ISD) and among different ISDs. ASes further propagate selected PCBs to their neighboring ASes. As a PCB traverses the network, it carries path segments, which can subsequently be used for traffic forwarding.

**Path Transparency**: Path transparency is a property of a network architecture that gives endpoints full visibility over the network paths their packets are taking. Path transparency is weaker than path control.

**SCMP**: SCION Control Message Protocol. SCMP is used for signaling connectivity problems, analogous to the Internet Control Message Protocol (ICMP). SCMP provides network diagnostic and error messages.

**Valley route**: A valley route contains ASes that do not profit economically from traffic on this route. The name comes from the fact that such routes go “down” (following parent-child links) before going “up” (following child-parent links).


## Conventions and Definitions

{::boilerplate bcp14-tagged}


## Overview

The SCION data plane forwards inter-domain packets between ASes. SCION routers are normally deployed at the edge of an AS, and peer with neighbor SCION routers. Inter-domain forwarding is based on end-to-end path information contained in the packet header. This path information consists of a sequence of hop fields (HFs). Each hop field corresponds to an AS on the path, and it includes an ingress- as well as an egress interface ID, which univocally identify the ingress and egress interfaces within the AS. The information is authenticated with a Message Authentication Code (MAC) to prevent forgery.

This concept allows SCION routers to forward packets to a neighbor AS without inspecting the destination address and also without consulting an inter-domain forwarding table. Intra-domain forwarding and routing are based on existing mechanisms (e.g., IP). A SCION border router reuses existing intra-domain infrastructure to communicate to other SCION routers or SCION endpoints within its AS. The last SCION router at the destination AS therefore uses the destination address to forward the packet to the appropriate local endpoint.

This SCION design choice has the following advantages:

- It provides control and transparency over forwarding paths to endpoints.
- It simplifies the packet-processing at routers: Instead of having to perform longest-prefix matching on IP addresses, which requires expensive hardware and substantial amounts of energy, a router can simply, after having verified the authenticity of the hop field's MAC, access the next hop from the packet header.


### Inter- and Intra-Domain Forwarding

As SCION is an inter-domain network architecture, it is not concerned with intra-domain forwarding. This corresponds to the general practice today where BGP and IP are used for inter-domain routing and forwarding, respectively, but ASes use an intra-domain protocol of their choice, for example OSPF or IS-IS for routing and IP, MPLS, and various layer-2 protocols for forwarding. In fact, even if ASes use IP-forwarding internally today, they typically encapsulate the original IP packet they receive at the edge of their network into another IP packet with the destination address set to the egress border router, to avoid full inter-domain forwarding tables at internal routers.

SCION emphasizes this separation, as SCION is used exclusively for inter-domain forwarding, and re-uses the intra-domain network fabric to provide connectivity among all SCION infrastructure services, border routers, and endpoints. As a consequence, minimal change to the infrastructure is required for ISPs when deploying SCION.

Although a complete SCION address is composed of the <ISD, AS, endpoint address> 3-tuple, the endpoint address is not used for inter-domain routing or forwarding. This implies that the endpoint addresses are not required to be globally unique or globally routable, they can be selected independently by the corresponding ASes. This means, for example, that an endpoint identified by a link-local IPv6 address in the source AS can directly communicate with an endpoint identified by a globally routable IPv4 address via SCION. Alternatively, it is possible for two SCION hosts with the same IPv4 address 10.0.0.42 but located in different ASes to communicate with each other via SCION ({{RFC1918}}).


### Intra-Domain Forwarding Process

The full forwarding process for a packet transiting an intermediate AS consists of the following steps.

**Note:** In this context, a border router is called **ingress** border router when it refers to an entrance border router to an AS, as seen from the direction of travel of the SCION packet. A border router is called **egress** border router when it refers to an exit border router of an AS, as seen from the direction of travel of the SCION packet.

1. The AS's SCION ingress router receives a SCION packet from the neighboring AS.
2. The SCION router parses, validates, and authenticates the SCION header.
3. The SCION router maps the egress interface ID in the current hop field of the SCION header to the destination "intra-protocol" address of the egress border router (where "intra-protocol" is the intra-domain forwarding protocol, e.g., MPLS or IP).
4. The packet is forwarded within the AS by routers and switches based on the "intra-protocol" header.
5. Upon receiving the packet, the SCION egress router strips off the "intra-protocol" header, again validates and updates the SCION header, and forwards the packet to the neighboring SCION router.

**Note:** The current SCION implementation uses the UDP/IP protocol as underlay. However, the use of other underlay protocols is possible and allowed.


## Path Construction (Segment Combinations) {#construction}

Paths are discovered by the SCION control plane, that makes them available to SCION endpoints in the form of path segments. As described in {{I-D.scion-cp}}, there are three kinds of path segments: up, down, and core. In the data plane, a SCION endpoint creates end-to-end paths from the path segments, by combining multiple path segments. Depending on the network topology, a SCION forwarding path can consist of one, two, or three segments. Each path segment contains several hop fields representing the ASes on the segment as well as one info field with basic information about the segment, such as a timestamp.

Segments cannot be combined arbitrarily. The source endpoint that constructs a forwarding path must obey the following rules:

- There can be at most one of each type of segment (up-, core-, and down-segment). Allowing multiple up- or down-segments would decrease efficiency and the ability of ASes to enforce path policies.
- If an up-segment is present, it must be the first segment in the path.
- If a down-segment is present, it must be the last segment in the path.
- If there are two path segments (one up- and one down-segment) that both announce the same peering link, then a shortcut via this peering link is possible.
- If there are two path segments (one up- and one down-segment) that share a common ancestor AS (in the direction of beaconing), then a shortcut via this common ancestor AS is possible.
- Additionally, all segments without any peering possibility need to consist of at least two hop fields.

**Note:** The type of segment is known to the endpoint but is not visible in the path header of data packets. A SCION router must therefore explicitly verify that these rules were followed correctly.

Besides enabling the enforcement of path policies, the above rules also protect the economic interest of ASes, as they prevent building “valley paths”. A valley path contains ASes that do not profit economically from traffic on this route. The name comes from the fact that such paths go “down” (following parent-child links) before going “up” (following child-parent links).

{{figure-1}} below shows the different allowed segment combinations.

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
{: #figure-1 title="Illustration of possible path-segment combinations. Each node represents a SCION Autonomous System."}


The following path-segment combinations are allowed:

- **Communication through core ASes**:

  - **Core-segment combination** (Cases 1a, 1b, 1c, 1d in the previous figure): The up- and down-segments of source and destination do not have an AS in common. In this case, a core-segment is required to connect the source's up- and the destination's down-segment (Case 1a). If either the source or the destination AS is a core AS (Case 1b) or both are core ASes (Cases 1c and 1d), then no up- or down-segment(s) are required to connect the respective AS(es) to the core-segment.
  - **Immediate combination** (Cases 2a, 2b in the figure): The last AS on the up-segment (which is necessarily a core AS) is the same as the first AS on the down-segment. In this case, a simple combination of up- and down-segments creates a valid forwarding path. In Case 2b, only one segment is required.

- **Peering shortcut** (Cases 3a and 3b): A peering link exists between the up- and down-segment. The extraneous path segments to the core are cut off. Note that the up- and down-segments do not need to originate from the same core AS and the peering link could also be traversing to a different ISD.
- **AS shortcut** (Cases 4a and 4b): The up- and down-segments intersect at a non-core AS. This is the case of a shortcut where an up-segment and a down-segment meet below the ISD core. In this case, a shorter path is made possible by removing the extraneous part of the path to the core. Note that the up- and down-segments do not need to originate from the same core AS and can even be in different ISDs (if the AS at the intersection is part of multiple ISDs).
- **On-path** (Case 5): In the case where the source's up-segment contains the destination AS or the destination's down-segment contains the source AS, a single segment is sufficient to construct a forwarding path. Again, no core AS is on the final path.


## Path Authorization

The SCION data plane provides *path authorization*. This property ensures that data packets always traverse the network using path segments that were explicitly authorized by the respective ASes, and prevents endpoints from constructing unauthorized paths or paths containing loops. SCION uses symmetric cryptography in the form of message-authentication codes (MACs) to authenticate the information encoded in hop fields. Such MACs are verified by routers at forwarding. For a detailed specification, see [](#path-auth).


# SCION Header Specification {#header}

This section provides a detailed specification of the SCION packet header. SCION also supports extension headers - these are described, too.


## SCION Packet Header Format {#format}

The SCION packet header is composed of a common header, an address header, a path header, and an optional extension header, see {{figure-2}} below.

~~~~
+--------------------------------------------------------+
|                     Common header                      |
|                                                        |
+--------------------------------------------------------+
|                       Addresses                        |
|                                                        |
+--------------------------------------------------------+
|                          Path                          |
|                                                        |
+--------------------------------------------------------+
|                 Extensions (optional)                  |
|                                                        |
+--------------------------------------------------------+
~~~~
{: #figure-2 title="High-level SCION header structure"}

The *common header* contains important meta information like a version number and lengths of the header and payload. In particular, it contains flags that control the format of subsequent headers such as the address and path headers. For more details, see [](#common-header).

The *address header* contains the ISD-, AS-, and endpoint-addresses of source and destination. The type and length of endpoint addresses are variable and can be set independently using flags in the common header. For more details, see [](#address-header).

The *path header* contains the full AS-level forwarding path of the packet. A path type field in the common header specifies the path format used in the path header. For more details, see [](#path-header).

Finally, the optional *extension* header contains a variable number of hop-by-hop and end-to-end options, similar to the extensions in the IPv6 header {{RFC8200}}. For more details, see [](#ext-header).


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
- `NextHdr`: Encodes the type of the first header after the SCION header. This can be either a SCION extension or a layer-4 protocol such as TCP or UDP. Values of this field respect the Assigned SCION Protocol Numbers (see [](#protnum).
- `HdrLen`: Specifies the entire length of the SCION header in bytes, i.e., the sum of the lengths of the common header, the address header, and the path header. All SCION header fields are aligned to a multiple of 4 bytes. The SCION header length is computed as `HdrLen` * 4 bytes. The 8 bits of the `HdrLen` field limit the SCION header to a maximum of 255 * 4 = 1020 bytes.
- `PayloadLen`: Specifies the length of the payload in bytes. The payload includes (SCION) extension headers and the L4 payload. This field is 16 bits long, supporting a maximum payload size of 65'535 bytes.
- `PathType`: Specifies the type of the SCION path. It is possible to specify up to 256 different types. The format of one path type is independent of all other path types. The currently defined SCION path types are Empty (0), SCION (1), OneHopPath (2), EPIC (3) and COLIBRI (4). This document only specifies the Empty, SCION and OneHopPath path types. The other path types are currently experimental.

| Value | Path Type                      |
|-------+--------------------------------|
| 0     | Empty path (`EmptyPath`)       |
| 1     | SCION (`SCION`)                |
| 2     | One-hop path (`OneHopPath`)    |
| 3     | EPIC path (experimental)       |
| 4     | COLIBRI path (experimental)    |
{: #table-1 title="Currently defined SCION path types"}

- `DT/DL/ST/SL`: These fields define the endpoint-address type and endpoint-address length for the source and destination endpoint. `DT` and `DL` stand for Destination Type and Destination Length, whereas `ST` and `SL` stand for Source Type and Source Length. The possible endpoint address length values are 4 bytes, 8 bytes, 12 bytes, and 16 bytes. If some address has a length different from the supported values, the next larger size can be used and the address can be padded with zeros. {{table-2}} below lists the currently used values for address length. The "type" identifier is only defined in combination with a specific address length. For example, address type "0" is defined as IPv4 in combination with address length 4, but in combination with address length 16, it stands for IPv6. Per address length, several sub-types are possible. {{table-3}} shows the currently valid allocations of type values to length values.

| DL/SL Value | Address Length |
|-------------+----------------|
| 0           | 4 bytes        |
| 1           | 8 bytes        |
| 2           | 12 bytes       |
| 3           | 16 bytes       |
{: #table-2 title="Address length values"}


| Length (bytes) | Type | Type/Length (binary) | Interpretation |
|----------------+------+----------------------+----------------|
| 4              | 0    | 0b0000               | IPv4           |
| 4              | 1    | 0b0100               | Service        |
| 16             | 0    | 0b1100               | IPv6           |
| other          |      |                      | Unassigned     |
{: #table-3 title="Allocations of type values to length values"}

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

**Note:** For more information on addressing in SCION, see the introduction of the SCION Control Plane Specification ({{I-D.scion-cp}}).


### Path Header {#path-header}

The path header of a SCION packet differs for each SCION path type.

**Note:** The path type is set in the `PathType` field of the SCION common header.

Currently, SCION supports three path types:

- The `Empty` path type (`PathType=0`). For more information, see [](#empty).
- The `SCION` path type (`PathType=1`). This is the standard path type in SCION. For a detailed description, see [](#scion-path-type).
- The `OneHopPath` path type (`PathType=2`). For more information, see [](#onehop).


#### Empty Path Type {#empty}

The `Empty` path type is used to send traffic within an AS. It has no additional fields, i.e., it consumes 0 bytes on the wire.

One use case of the `Empty` path type lies in the context of link-failure detection. To this end, SCION uses the Bidirectional Forwarding Detection (BFD) protocol ({{RFC5880}} and {{RFC5881}}). BFD is a protocol intended to detect faults in the bidirectional path between two forwarding engines, with typically very low latency. It operates independently of media, data protocols, and routing protocols. SCION uses the `Empty` path type, together with the [](#onehop), to bootstrap BFD within SCION.


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


It consists of a path meta header, up to 3 info fields and up to 64 hop fields.

- The path meta header (`PathMetaHdr`) indicates the currently valid info field and hop field while the packet is traversing the network along the path, as well as the number of hop fields per segment.
- The number of info fields (`InfoField`) equals the number of path segments that the path contains - there is one info field per path segment. Each info field contains basic information about the corresponding segment, such as a timestamp indicating the creation time. There are also two flags. One specifies whether the segment must be traversed in construction direction, the other whether the first or last hop field in the segment represents a peering hop field.
- Each hop field (`HopField`) represents a hop through an AS on the path, with the ingress and egress interface identifiers for this AS. This information is authenticated with a Message Authentication Code MAC to prevent forgery.

The SCION header is created by extracting the required info fields and hop fields from the corresponding path segments. The process of extracting is illustrated in {{figure-6}} below. Note that ASes at the joints of multiple segments are represented by two hop fields. Be aware that these hop fields are not equal! In the hop field that represents the last hop in the first segment (seen in the direction of travel), only the ingress interface will be specified. However, in the hop field that represents the first hop in the second segment (also in the direction of travel), only the egress interface will be defined. Thus, the two hop fields for this one AS build a full hop through the AS, specifying both the ingress and egress interface. As such, they bring the two adjacent segments together.

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



##### Path Meta Header Field {#pathmetahdr}

The 4-byte Path Meta Header field (`PathMetaHdr`) defines meta information about the SCION path that is contained in the path header. It has the following format:

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| C |  CurrHF   |    RSV    |  Seg0Len  |  Seg1Len  |  Seg2Len  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #figure-7 title="SCION path type - Format of the Path Meta Header field"}


- `C` (urrINF): Specifies a 2-bits index (0-based) pointing to the current info field for the packet on its way through the network (see [](#offset-calc) below).
- `CurrHF`: Specifies a 6-bits index (0-based) pointing to the current hop field for the packet on its way through the network (see [](#offset-calc) below). Note that the `CurrHF` index must point to a hop field that is part of the current path segment, as indicated by the `CurrINF` index.

Both indices are used by SCION routers when forwarding data traffic through the network. The SCION routers also increment the indexes if required. For more details, see [](#process-router).

- `Seg{0,1,2}Len`: The number of hop fields in a given segment. Seg{i}Len > 0 implies that segment *i* contains at least one hop field, which means that info field *i* exists. (If Seg{i}Len = 0 then segment *i* is empty, meaning that this path does not include segment *i*, and therefore there is no info field *i*.) The following rules apply:

  - The total number of hop fields in an end-to-end path must equal the sum of all `Seg{0,1,2}Len` contained in this end-to-end path.
  - It is an error to have Seg{X}Len > 0 AND Seg{Y}Len == 0, where 2 >= *X* > *Y* >= 0. That is, if path segment Y is empty, the following path segment X must also be empty.

- `RSV`: Unused and reserved for future use.


##### Path Offset Calculations {#offset-calc}

The path offset calculations are used by the SCION border routers to determine the info field and hop field that are currently valid for the packet on its way through the network.

The following rules apply when calculating the path offsets:

~~~~
   if Seg2Len > 0: NumINF = 3
   else if Seg1Len > 0: NumINF = 2
   else if Seg0Len > 0: NumINF = 1
   else: invalid
~~~~

The offsets of the current info field and current hop field (relative to the end of the address header) are now calculated as:

~~~~
   B = byte
   InfoFieldOffset = 4B + 8B.CurrINF
   HopFieldOffset = 4B + 8B.NumINF + 12B.CurrHF
~~~~

To check that the current hop field is in the segment of the current info field, the `CurrHF` needs to be compared to the `SegLen` fields of the current and preceding info fields.


##### Info Field {#inffield}

kjhyxdbc


##### Hop Field {#hopfld}

kjhyxdbc


#### One-Hop Path Type {#onehop}

kjhyxdbc


#### Path Reversal {#reverse}

kjhyxdbc



## Extension Headers {#ext-header}

cvcxvbc


### Format of the SCION Options Headers {#oh-format}

xccxy


#### Options Field {#optfld}

nnm



##### Pad1 Option {#pad1}

Alignment requirement: none.


##### PadN Option {#padn}

Alignment requirement: none.


## Upper-Layer Protocol Issues


### Pseudo Header for Upper-Layer Checksum {#pseudo}

xccxy





# Life of a SCION Data Packet

abcc


# Path Authorization {#path-auth}

abcc



# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.


# Assigned SCION Protocol Numbers {#protnum}
{:numbered="false"}

abcd
