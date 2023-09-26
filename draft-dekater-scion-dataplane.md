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


#### Intra-Domain Forwarding Process

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

abcc


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
