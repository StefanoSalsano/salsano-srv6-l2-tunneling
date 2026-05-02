---
title: "SRv6 Layer-2 Tunneling with IPv6 Source-Address-Based Service Identification"
abbrev: "SRv6 L2 Tunneling"
category: exp

docname: draft-salsano-srv6-l2-tunneling-latest
submissiontype: independent
ipr: trust200902
# area: AREA
# workgroup: SIG on EIP
keyword:
 - SRv6
 - IPv6
 - Layer 2
 - Tunneling
 - VXLAN
venue:
#  group: EIP
#  type: SIG
#  mail: eip@cnit.it
#  arch: http://postino.cnit.it/cgi-bin/mailman/private/eip/
  github: "StefanoSalsano/salsano-srv6-l2-tunneling"
  latest: "https://StefanoSalsano.github.io/salsano-srv6-l2-tunneling/draft-salsano-srv6-l2-tunneling.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  - name: "Stefano Salsano"
    ins: "S. Salsano"
    organization: "Univ. of Rome Tor Vergata / CNIT"
    email: "stefano.salsano@uniroma2.it"
  - name: "Andrea Mayer"
    ins: "A. Mayer"
    organization: "Univ. of Rome Tor Vergata / CNIT / Common Net"
    email: "andrea.mayer@uniroma2.it"

normative:
  RFC8200:
  RFC8754:
  RFC8986:

informative:
  RFC3704:
  RFC7348:
  RFC8704:
  RFC8799:
  RFC9252:
  RFC9800:
  RFC9819:
  I-D.cheng-spring-srv6-encoding-network-sliceid:
  I-D.ietf-6man-vpn-dest-opt:
  I-D.li-6man-apn-ipv6-encap:
  HYDN-MAITI-2026:
    title: "SRv6 Layer-2 Overlays with VXLAN-like Semantics: Linux End.DT2U, the sr6 Device, and Scalable Service Identification"
    author:
      - name: "Stefano Salsano"
      - name: "Andrea Mayer"
      - name: "Ahmed Abdelsalam"
      - name: "Clarence Filsfils"
    date: 2026

--- abstract

SRv6 defines Layer-2-oriented endpoint behaviors and supports
SRv6-based Layer-2 overlay services. However, practical Layer-2
tunneling over SRv6 still lacks a simple and efficient service
identification model comparable to the VXLAN VNI. This limitation is
particularly relevant in uSID-based deployments, where Destination
Address space is a scarce resource and cannot be consumed freely for
per-service identification.

This document proposes an SRv6 Layer-2 tunneling approach in which a
24-bit Layer-2 service identifier is encoded in the 24 least
significant bits of the outer IPv6 Source Address, while the
remaining 104 bits preserve normal IPv6 Source Address semantics.
This preserves Destination Address space for SRv6 steering while
enabling VXLAN-like identification of Layer-2 overlay services. The
mechanism is intended for deployment within a single administrative
limited domain.

--- middle

# Introduction

Segment Routing over IPv6 (SRv6) {{RFC8754}} {{RFC8986}} defines
endpoint behaviors that support Layer-2 forwarding and overlay
services. However, SRv6 currently lacks a compact and operationally
explicit service identifier for Layer-2 overlays comparable to the
24-bit VXLAN Network Identifier (VNI). Encoding such an identifier
in the SRv6 Destination Address scales poorly, particularly in
deployments based on compressed SID representations such as uSID,
where the SID space available after the locator is intrinsically
limited.

This document defines an SRv6 Layer-2 tunneling approach in which a
24-bit Layer-2 service identifier is encoded in the 24 least
significant bits of the outer IPv6 Source Address. The outer IPv6
Destination Address continues to identify the remote endpoint and
the SRv6 behavior to be executed, following normal SRv6 processing,
while the outer IPv6 Source Address carries a compact, VXLAN-like
service identifier without consuming Destination Address space.
The remaining 104 bits of the outer IPv6 Source Address preserve
normal IPv6 Source Address semantics, so that the resulting address
remains a valid unicast IPv6 address as required by {{RFC8200}}.
The size of the proposed identifier matches the 24-bit VXLAN
Network Identifier, so that existing operational and information
models that already manage Layer-2 services through 24-bit VNIs
can be reused over SRv6 without modification.

The mechanism defined in this document is intended for deployment
within a limited domain, in the sense of {{RFC8799}}, that is,
within a set of nodes and links under a single administrative
authority where consistent treatment of the proposed Source Address
encoding can be ensured by the operator. The applicability and the
operational considerations associated with this scoping are
discussed in {{applicability}}.

# Scenario and Motivation

This section describes the deployment context that motivates the
mechanism defined in this document.

Modern Layer-2 overlay deployments, in particular in datacenter
networks supporting hyperscale and AI-oriented workloads, rely on
encapsulation technologies that combine a clear separation between
tunnel-endpoint identification and bridge-domain identification
with a compact and explicit identifier for each bridge domain.
VXLAN {{RFC7348}} is the most widespread example of this
operational model: the outer IP addressing identifies the remote
tunnel endpoint, while a 24-bit VXLAN Network Identifier (VNI),
carried in a dedicated header field outside the outer IP addressing,
identifies the specific bridge domain. This separation
has made VXLAN the de facto reference for scalable Ethernet
overlays.

SRv6 already defines Layer-2 endpoint behaviors in {{RFC8986}},
including `End.DT2U` for unicast Layer-2 table lookup and
`End.DT2M` for broadcast/unknown-unicast/multicast (BUM) flooding
in a local Ethernet bridge domain. Together, these behaviors
provide the architectural basis for SRv6 Layer-2 overlays. The signaling of these behaviors as Service SIDs over
BGP, in particular for SRv6-based EVPN services, is specified in
{{RFC9252}} and {{RFC9819}}. In current SRv6 practice, following
this BGP overlay services model, the identification of the
specific bridge domain is typically absorbed into the outer IPv6
Destination Address, for example through a service uSID at the
egress node, so that endpoint identification, behavior selection,
and bridge-domain identification share the same address.

The pressure on the Destination Address has further increased with
the introduction of compressed SRv6 segment-list encoding mechanisms
{{RFC9800}}, such as uSID, in which the SID space available after
the locator portion of the address is intrinsically limited.
Allocating a 24-bit VXLAN-like service identifier directly in the
Destination Address would consume a significant fraction of the
available SID space and would couple service identification with
locator and behavior encoding in an undesirable way.

For these reasons, this document defines an SRv6-native Layer-2
service identification mechanism that preserves the Destination
Address for SRv6 endpoint identification, behavior selection, and
path steering, while providing a compact 24-bit service identifier
comparable in size and role to the VXLAN VNI. This is achieved by
encoding the service identifier in the 24 least significant bits of
the outer IPv6 Source Address. The conceptual relationship between
the VXLAN model, current SRv6 practice, and the approach proposed
in this document is illustrated in {{fig-overview}}.

~~~
   VXLAN model            Current SRv6           This document
   -----------            ------------           -------------

   outer IP DA            outer IPv6 DA          outer IPv6 DA
   -> remote VTEP         -> egress node +       -> egress node +
                             SRv6 behavior          SRv6 behavior
                             + service uSID

   VXLAN VNI (24b)        (bridge-domain id      outer IPv6 SA
   -> bridge domain       absorbed into the      lower 24 bits
                          Destination Address)   -> bridge domain
~~~
{: #fig-overview title="Service identification: VXLAN, current SRv6, and this document."}

A practical consequence of this design is feature parity with the
VXLAN VNI in terms of identifier size and role. Cloud orchestration
systems and datacenter virtualization platforms, such as OpenStack
Neutron and Apache CloudStack, today manage Layer-2 services
through 24-bit VNIs that are stored, signaled, and validated as
24-bit values across their information models, APIs, and
configuration databases. The mechanism proposed in this document
defines a 24-bit SRv6-native Layer-2 service identifier that can
be mapped one-to-one to such a VNI. This enables SRv6 to be
adopted as the underlay encapsulation technology in environments
currently based on VXLAN without requiring changes to the
orchestration information model, the service-instance APIs, or
the operational tooling that depends on the 24-bit identifier
space.

Implementation aspects of an SRv6 Layer-2 overlay using this
service-identification model, including integration with the local
Layer-2 data plane, are discussed in {{HYDN-MAITI-2026}}.

# Problem Statement and Design Goals

SRv6 Layer-2 tunneling requires two distinct functions. First, a
packet must be delivered to the remote node that performs the
decapsulation behavior. Second, the decapsulating node must
identify the specific bridge domain to which the inner frame
belongs.

As discussed in {{fig-overview}}, VXLAN separates the two
functions cleanly through the outer IP addressing and the 24-bit
VNI. In SRv6, the outer IPv6 Destination Address and, more
generally, the SID list are naturally used for endpoint
identification and behavior selection, but bridge-domain
identification is typically absorbed into the same Destination
Address. Encoding a VXLAN-like 24-bit identifier in the
Destination Address is impractical in deployments based on
compressed SID representations such as uSID {{RFC9800}}, because
it would consume a significant fraction of the available SID
space.

As a result, SRv6 Layer-2 tunneling lacks a compact and
implementation-friendly bridge-domain identification mechanism
with feature parity to VXLAN. A practical solution should
therefore satisfy the following design goals:

* preserve the Destination Address primarily for SRv6 endpoint
  identification and behavior selection;
* provide a compact service identifier comparable in size and role to
  the VXLAN VNI;
* enable one-to-one reuse of the 24-bit Layer-2 service identifiers
  managed by existing VXLAN-based cloud orchestration systems,
  without changes to their underlying information models;
* avoid excessive consumption of SID space, especially in uSID-based
  deployments;
* fit naturally into an SRv6 encapsulation model;
* be simple to implement in practical dataplanes.

This document addresses that problem by proposing to encode a 24-bit
service identifier in the outer IPv6 Source Address.

# Tunnel Identification in VXLAN and Current SRv6

VXLAN and SRv6 both provide a way to deliver packets to a remote tunnel
endpoint and to invoke a decapsulation function at that endpoint.
However, they differ significantly in how they identify the specific
Layer-2 service associated with a tunneled frame.

## Tunnel Identification in VXLAN

In VXLAN, summarized in {{fig-overview}}, the outer IP header
identifies the remote tunnel endpoint and the 24-bit VXLAN Network
Identifier (VNI) identifies the specific bridge domain, in a
dedicated header field independent of the outer IP addressing.

## Tunnel Identification in Current SRv6

When SRv6-based BGP overlay services are deployed using compressed
SID representations such as uSID {{RFC9800}}, the decapsulating
tunnel endpoint is typically identified by a local Service SID,
signaled over BGP as defined in {{RFC9252}} and {{RFC9819}}.

A uSID list commonly ends with:

* a locator uSID, which identifies the egress or decapsulation node; and
* one or more service uSIDs, which identify the behavior to be executed
  and the specific service instance at that node.

For example, a service uSID may identify:

* a specific VRF in a decapsulation behavior such as End.DTx;
* a specific Layer-2 tunnel context, e.g. through behaviors such as
  `End.DT2U`, `End.DT2M`, `End.DX2`, or `End.DX2V` as encoded in the
  SRv6 L2 Service TLV of the BGP Prefix-SID attribute defined in
  {{RFC9252}};
* a specific routing adjacency; or
* another local service instance bound to the node.

In this model, the IPv6 Destination Address is used not only to steer
the packet to the correct node, but also to identify the local service
instance to be applied at the endpoint.

This approach is workable, but it has an important limitation. The
SRv6 SID structure defined in {{RFC9252}} splits the SID into
Locator Block, Locator Node, Function, and Argument fields, with
sizes signaled by the SRv6 SID Structure Sub-Sub-TLV and bounded
by the 128-bit address. With a typical 2-octet uSID granularity,
i.e. a Function Length of 16 bits, the service-uSID space available
at a node is on the order of 2^16 values. This space must be shared
among all local service instances of that node, cumulatively
including VRFs, Layer-2 tunnels, routing adjacencies, and other
local behaviors. Larger Function or Argument lengths are possible,
but they consume additional bits of the SID and are constrained by
the locator allocation and by interoperability considerations such
as MPLS-Label-field transposition.

As a consequence, the same limited service-uSID space is used
both to identify the behavior and to distinguish among all
concrete service instances supported by the node. In particular,
allocating a VXLAN-like 24-bit bridge-domain identifier directly
in the SRv6 Destination Address is impractical in uSID-based
deployments, because it would consume a substantial fraction of
the available SID space.

This is a key difference from VXLAN. In VXLAN, the bridge-domain
identifier is carried in a dedicated field outside the outer IP
addressing. In SRv6, including the BGP overlay services model of
{{RFC9252}}, bridge-domain identification is typically absorbed
into the Destination Address semantics, which makes scalable
Layer-2 tunnel identification more difficult.

# Source-Address-Based Service Identification {#sa-svc-id}

This document proposes to encode the Layer-2 service identifier in the
24 least significant bits of the outer IPv6 Source Address.

In this document, the Layer-2 service unit identified by the
24-bit value carried in `SA[23:0]` is consistently referred to as
a *bridge domain*. This term encompasses both unicast Layer-2
forwarding and broadcast/unknown-unicast/multicast (BUM) flooding
semantics at the egress, depending on the SRv6 behavior selected
through the Destination Address, as discussed in {{relation}}.

The key idea is to separate the two functions that, in current SRv6
practice, are both absorbed into the Destination Address semantics:

* identification of the remote decapsulation node and of the SRv6
  behavior to be executed; and
* identification of the specific bridge domain associated with the
  tunneled frame.

In the approach proposed here, the outer IPv6 Destination Address
continues to identify the remote endpoint and the SRv6 decapsulation
behavior, following normal SRv6 processing. The outer IPv6 Source
Address is instead used to carry a 24-bit service identifier,
functionally similar to the VXLAN VNI.

More precisely, let `SA` denote the 128-bit outer IPv6 Source Address.
This document defines the service identifier as:

~~~
SERVICE_ID = SA[23:0]
~~~

that is, the 24 least significant bits of the outer IPv6 Source Address.

The remaining upper 104 bits of the Source Address, i.e. `SA[127:24]`,
MUST preserve the normal semantics of the IPv6 Source Address. In
particular, they MUST be assigned so that the source address remains
meaningful and reachable in the IPv6 domain where the SRv6 tunnel is
used. This is important to preserve basic IPv6 operational properties,
including the ability to receive return traffic and to support
operations such as ICMPv6 echo reply processing. In other words, the use
of the 24 least significant bits for service identification MUST NOT
turn the outer IPv6 Source Address into a purely opaque field with no
valid source-address semantics.

The upper 104 bits `SA[127:24]` are a regular IPv6 prefix assigned
to the ingress node, in the sense of {{RFC8200}}. They are not an
SRv6 SID and MUST NOT be interpreted as a uSID container or
otherwise subjected to compressed-SID processing such as that
defined in {{RFC9800}}.

Using the 24 least significant bits of the Source Address provides a
compact and explicit service identifier without consuming bits from the
SRv6 Destination Address. This is particularly beneficial in uSID-based
deployments, where Destination Address space is a scarce resource and
should be preserved for locator encoding, endpoint identification,
behavior selection, and path steering.

At the decapsulating node, the SRv6 Destination Address identifies the
local decapsulation behavior, while the 24-bit value extracted from the
outer IPv6 Source Address identifies the specific Layer-2 service
instance. In this way, the proposed solution provides a separation
between tunnel-endpoint identification and service identification that
is similar, in functional terms, to the separation between outer IP
addressing and VNI in VXLAN.

The 24-bit service identifier carried in the Source Address
identifies the bridge domain at the egress node to which the
inner Ethernet frame is to be delivered.

This document does not mandate a specific control-plane signaling
mechanism for the 24-bit service identifier. Such mechanisms are outside
the scope of this document and may be defined by future specifications.

The proposed use of the Source Address does not alter the role of the
Destination Address in SRv6 forwarding. Instead, it complements it by
providing a separate field for compact Layer-2 service identification.

# Relation to Existing SRv6 Behaviors {#relation}

The mechanism proposed in this document is intended to define two
new SRv6 Layer-2 decapsulation behaviors, denoted as `End.DT2U.SA`
and `End.DT2M.SA`, rather than to redefine the existing Layer-2
endpoint behaviors of {{RFC8986}}.

Among the Layer-2 behaviors defined in {{RFC8986}}, `End.DT2U` and
`End.DT2M` provide the closest functional reference for VXLAN-like
Ethernet overlays. `End.DT2U` performs decapsulation of an SRv6
packet carrying an inner Ethernet frame and delivers the frame
into a local Ethernet bridge domain through unicast Layer-2 table
lookup. `End.DT2M` performs decapsulation and delivery of the
inner Ethernet frame to the same kind of bridge domain through
broadcast, unknown-unicast, and multicast (BUM) flooding
semantics, as commonly required to support traffic such as ARP,
unknown-MAC frames, and Layer-2 multicast in distributed Layer-2
overlays. Following the BGP overlay services framework specified
in {{RFC9252}} and {{RFC9819}}, the specific bridge domain
associated with the decapsulating node is typically identified
through the Destination Address semantics, for example by using a
service uSID encoded in the Function (and optionally Argument)
portion of the Service SID.

The `End.DT2U.SA` and `End.DT2M.SA` behaviors follow the same
overall model of Layer-2 decapsulation and delivery into a local
bridge domain as `End.DT2U` and `End.DT2M`, respectively, but
introduce an additional service-identification function. The
egress node and the SRv6 behavior are identified through the outer
IPv6 Destination Address, while the specific Layer-2 service
instance, i.e. the bridge domain, is identified by the 24-bit
value carried in the least significant bits of the outer IPv6
Source Address, as defined in this document.

For this reason, `End.DT2U.SA` can be viewed as an enhanced variant
of `End.DT2U`, and `End.DT2M.SA` as an enhanced variant of
`End.DT2M`, in which the bridge-domain identifier is carried
separately from the Destination Address. This provides a clearer
separation between endpoint identification and bridge-domain
identification, and avoids consuming Destination Address SID
space for VXLAN-like service identification.

The relationship between `End.DT2M.SA` and `End.DT2U.SA` mirrors
exactly the relationship between `End.DT2M` and `End.DT2U` in
{{RFC8986}}. The two behaviors operate on the same set of bridge
domains at the egress node, identified consistently through the
24-bit value `SA[23:0]`, and differ only in the forwarding
semantics applied to the inner Ethernet frame after decapsulation,
i.e. unicast Layer-2 table lookup for `End.DT2U.SA` versus
broadcast/flooding for `End.DT2M.SA`. Consequently, the
encapsulation, the source-address-based service-identification
mechanism, the applicability constraints, and the deployment
considerations defined in this document apply identically to both
behaviors. Throughout this document, statements about
`End.DT2U.SA` apply equally to `End.DT2M.SA`, except where
explicitly distinguished.

The control plane mechanisms and BUM tunnel setup procedures
needed to instantiate Layer-2 BUM distribution across an SRv6
overlay, including ingress replication and multicast underlay
distribution as commonly used in VXLAN/EVPN deployments, are
inherited from the existing SRv6 BGP overlay services framework
{{RFC9252}}. They are independent of whether the bridge domain is
identified by a Service SID encoded in the Destination Address
(per {{RFC9252}}) or by a 24-bit identifier encoded in the Source
Address (per this document), and are therefore outside the scope
of this document.

The Layer-2 cross-connect behaviors `End.DX2` and `End.DX2V` of
{{RFC8986}}, which forward the decapsulated frame toward a
specific outgoing Layer-2 interface or VLAN, are functionally
distinct from `End.DT2U`/`End.DT2M` because they do not perform
bridge-domain Layer-2 forwarding. The cardinality of the service
instances supported by these cross-connect behaviors at a given
egress node is bounded by the number of locally configured
Layer-2 interfaces and, for `End.DX2V`, by the number of VLANs
per interface, which is typically well within the order of 2^16
service-uSID values made available by a 16-bit Function Length in
the SRv6 SID structure of {{RFC9252}}. An operator can readily
reserve a contiguous range within that space for the
identification of all Layer-2 interfaces (or VLAN-tagged
sub-interfaces) of a node. As a consequence, a standard Service
SID encoded in the Destination Address per {{RFC9252}} remains
operationally adequate for `End.DX2` and `End.DX2V`, and the
source-address-based encoding defined in this document is not
required for these behaviors. They are therefore not the primary
reference for the mechanism defined in this document.

This document therefore assumes that the proposed mechanism is
specified as two distinct SRv6 behaviors, namely `End.DT2U.SA` and
`End.DT2M.SA`, rather than as a backward-compatible
reinterpretation of existing Layer-2 behaviors.

# Encapsulation Procedure

At the ingress node, the encapsulating tunnel endpoint receives an
Ethernet frame to be transported over an SRv6 Layer-2 tunnel and
performs the following steps:

1. determine the remote decapsulation endpoint and the
   corresponding Service SID, which encodes either the
   `End.DT2U.SA` behavior, when the inner frame is to be delivered
   as unicast at the egress, or the `End.DT2M.SA` behavior, when
   the inner frame is to be delivered as BUM (broadcast,
   unknown-unicast or multicast) at the egress; this Service SID
   is encoded in the outer IPv6 Destination Address according to
   {{RFC8986}}, possibly in compressed form per {{RFC9800}}, and
   may be signaled over BGP per {{RFC9252}} and {{RFC9819}};
2. determine the 24-bit service identifier associated with the
   Layer-2 bridge domain; the mapping between bridge domains and
   24-bit identifiers is outside the scope of this document, and
   the same 24-bit value is used consistently for both
   `End.DT2U.SA` and `End.DT2M.SA`;
3. construct the outer IPv6 Source Address as defined in
   {{sa-svc-id}}, that is, with `SA[23:0]` carrying the service
   identifier and `SA[127:24]` carrying the ingress node's
   allocated IPv6 prefix;
4. encapsulate the original Ethernet frame as the inner payload of
   the resulting IPv6 packet, optionally including a Segment
   Routing Header {{RFC8754}} as required by the SRv6 policy in
   use;
5. forward the packet according to normal SRv6 processing.

The ingress node MUST NOT modify the inner Ethernet frame except
as required by normal tunnel processing.

A concrete example of the resulting encapsulated packet is shown
in {{fig-pkt-example}}, with the following sample allocations:

* Ingress node IPv6 prefix: 2001:db8:a:1::/104
* 24-bit Layer-2 service identifier (bridge domain): 0x010203
* Egress Service SID encoded in the Destination Address,
  illustrating either the `End.DT2U.SA` behavior or the
  `End.DT2M.SA` behavior (illustrative value:
  2001:db8:b:1:5300:0001::)

~~~
+---------------------------------------------------+
| Outer IPv6 Header                                 |
|   Source Address = 2001:db8:a:1::1:0203           |
|     SA[127:24] = 2001:db8:a:1::/104 (ingress)     |
|     SA[23:0]   = 0x010203 (SERVICE_ID)            |
|   Destination Address = Service SID identifying   |
|     either End.DT2U.SA or End.DT2M.SA behavior    |
|     (e.g., 2001:db8:b:1:5300:0001::)              |
+---------------------------------------------------+
| (Optional) Segment Routing Header                 |
+---------------------------------------------------+
| Inner Ethernet frame                              |
| (original Layer-2 payload)                        |
+---------------------------------------------------+
~~~
{: #fig-pkt-example title="Encapsulated packet structure with sample values."}

# Decapsulation Procedure

At the egress node, when an encapsulated packet is received and
the SRv6 active SID identifies either the `End.DT2U.SA` or the
`End.DT2M.SA` behavior, the node performs the following steps:

1. extract the 24-bit service identifier from the least
   significant bits of the outer IPv6 Source Address:
   `SERVICE_ID = SA[23:0]`;
2. map the extracted service identifier to the local Layer-2
   bridge domain to which the inner Ethernet frame is to be
   delivered;
3. remove the outer SRv6 encapsulation;
4. deliver the decapsulated Ethernet frame to the identified
   bridge domain, applying the forwarding semantics that
   correspond to the SRv6 behavior identified by the active SID:

   * for `End.DT2U.SA`, perform unicast Layer-2 table lookup in
     the bridge domain on the destination MAC address of the
     inner frame, as in `End.DT2U` of {{RFC8986}};
   * for `End.DT2M.SA`, perform broadcast/flooding in the bridge
     domain, as in `End.DT2M` of {{RFC8986}}.

If the extracted service identifier does not correspond to a valid
local Layer-2 bridge domain, the packet MUST be discarded.

The detailed error handling, OAM behavior, and optional ICMP
reporting for this case are left for future versions of this
document.


# Related Work {#related-work}

The encoding of service-identification or context information in
the IPv6 packet header has been explored in several IETF
specifications and ongoing work, using different mechanisms.

A directly comparable design pattern is proposed in
{{I-D.cheng-spring-srv6-encoding-network-sliceid}}, which encodes a
network slice identifier in the least significant bits of the
outer IPv6 Source Address of an SRv6 packet. That work shares the
same architectural choice adopted in this document, namely
encoding a compact identifier in the lower portion of the Source
Address while preserving normal IPv6 semantics in the upper bits.
The two specifications address different problems, network slicing
on one side and Layer-2 service identification on the other, but
they apply a consistent design pattern to the SRv6 Source Address.

Other related work uses different fields of the IPv6 packet header
to convey service or context information. The most directly
relevant baseline is the SRv6 BGP overlay services framework
defined in {{RFC9252}} and complemented by {{RFC9819}}, which
specifies how Service SIDs for L2 and L3 services, including
behaviors such as `End.DX2`, `End.DX2V`, `End.DT2U`, and
`End.DT2M`, are signaled in the BGP Prefix-SID attribute and
encoded into the outer IPv6 Destination Address according to the
SRv6 network programming model of {{RFC8986}}, possibly in
compressed form {{RFC9800}}. In this model, both the endpoint
and the specific bridge domain are identified through
the Destination Address. The IPv6 VPN Service Destination Option
{{I-D.ietf-6man-vpn-dest-opt}} instead encodes VPN identification
in an IPv6 Destination Option, that is, in an extension header
rather than in an address field. Application-aware IPv6 Networking
{{I-D.li-6man-apn-ipv6-encap}} similarly relies on extension-header
encapsulation to carry application-aware attributes.

These approaches differ from the mechanism defined in this document
in two main ways. First, they either consume Destination Address
space or require the addition of an IPv6 extension header to carry
the service identifier, whereas the mechanism defined here reuses
existing space in the outer IPv6 Source Address. Second, they
target different service models, in particular IPv6 VPNs and
application-aware networking, whereas this document focuses on
Layer-2 service identification for SRv6 Layer-2 tunnels.

# Design Alternatives Considered {#alternatives}

Several alternative approaches to encoding a 24-bit Layer-2 service
identifier in an SRv6 packet were considered during the design of
the mechanism defined in this document. Each is summarized below
together with the reason why it was not selected.

* IPv6 Flow Label.
  The 20-bit Flow Label of the outer IPv6 header could in
  principle carry a service identifier. However, 20 bits do not
  provide feature parity with the 24-bit VXLAN VNI; furthermore,
  the Flow Label is also used by the underlay for ECMP-related
  entropy and load balancing, and overloading it with
  service-identification semantics would conflict with this
  established use.

* Segment Routing Header TLV.
  A Type-Length-Value field in the SRH could carry the service
  identifier. This approach requires the presence of an SRH on
  every encapsulated packet, including in deployments that would
  otherwise not need one, and increases packet header size and
  parsing complexity in fast-path implementations. It also makes
  the service identifier accessible only after SRH parsing,
  complicating ingress-node-only forwarding decisions.

* IPv6 Hop-by-Hop or Destination Options.
  A new IPv6 extension header option could carry the service
  identifier, similarly to the IPv6 VPN Service Destination
  Option {{I-D.ietf-6man-vpn-dest-opt}} and to Application-aware
  IPv6 Networking {{I-D.li-6man-apn-ipv6-encap}}. While viable,
  this approach adds an additional header to every encapsulated
  packet and is known to be problematic in some deployment
  contexts due to inconsistent extension-header processing along
  Internet paths, even though the present mechanism is scoped to
  a limited domain.

* VXLAN-GPE inside SRv6.
  A VXLAN-GPE encapsulation could be carried as the inner
  payload of an SRv6 tunnel, preserving the original VNI
  semantics. This approach achieves feature parity with VXLAN at
  the cost of double encapsulation, with the corresponding
  overhead, increased packet size, and reduced effectiveness of
  compressed-SID encodings.

* Service identifier in the outer IPv6 Destination Address.
  The 24-bit service identifier could be encoded in the Function
  or Argument portion of the SRv6 Service SID, as in
  {{RFC9252}}. This is the existing baseline that the present
  mechanism is designed to avoid, because it consumes
  Destination Address space that is particularly scarce in
  uSID-based deployments {{RFC9800}}.

* Service identifier in the upper bits of the outer IPv6 Source
  Address.
  The 24-bit identifier could in principle be encoded in
  higher-order bits of the Source Address rather than in the
  least significant 24 bits. This was considered and rejected
  because it would interfere with prefix-based routing and
  aggregation of the ingress node prefix, and would not preserve
  the property that the upper bits remain a regular IPv6 prefix
  assigned to the ingress node.

The mechanism defined in this document, namely encoding the 24-bit
service identifier in the 24 least significant bits of the outer
IPv6 Source Address, was selected because it provides feature
parity with the 24-bit VXLAN VNI, does not consume Destination
Address space, does not require an additional extension header,
preserves a regular IPv6 prefix in the upper 104 bits of the
Source Address, and is straightforward to implement in dataplanes
that already handle SRv6 encapsulation.

# Applicability and Deployment Considerations {#applicability}

The mechanism defined in this document is applicable within a
limited domain, in the sense of {{RFC8799}}, under a single
administrative authority. Within such a domain, the operator can
ensure consistent allocation of IPv6 Source Address prefixes, the
encoding of the 24-bit service identifier in the lower bits of the
Source Address, and the corresponding configuration of the
decapsulation behavior at the egress nodes.

Packets carrying the proposed Source Address encoding SHOULD NOT
leave the limited domain in which the encoding is interpreted.
Nodes at the boundary of the limited domain SHOULD prevent egress
of packets that rely on this encoding for service identification,
either by terminating the SRv6 Layer-2 tunnels at the boundary or
by applying appropriate filtering policies.

## Source Address Allocation

To support the proposed encoding, the operator deploying this
mechanism allocates an IPv6 Source Address prefix to each ingress
node that performs Layer-2 encapsulation according to this
document. A natural allocation is a /104 prefix per ingress node,
or a shorter prefix shared among multiple ingress nodes, so that
the lower 24 bits remain available to encode the Layer-2 service
identifier.

The chosen prefix MUST be routable within the limited domain so
that the resulting outer IPv6 Source Address is a valid unicast
address of the ingress node, in the sense of Section 4.1 of
{{RFC8200}}, and so that return traffic and ICMPv6 messages
addressed to the Source Address can be delivered to the ingress
node.

## Source Address Validation and Ingress Filtering

The mechanism defined in this document interacts with Source
Address Validation (SAV) techniques such as ingress filtering
{{RFC3704}} and enhanced feasible-path uRPF {{RFC8704}}. Within
the limited domain, the operator MUST ensure that the routing
configuration and the SAV policies are consistent with the use of
the allocated Source Address prefixes by the ingress nodes. In
particular, packets sourced by an ingress node from any address
within its allocated prefix, including those carrying a non-zero
service identifier in the lower 24 bits, MUST NOT be discarded by
SAV mechanisms internal to the domain.

At the boundary of the limited domain, normal SAV policies may be
applied to traffic that leaves or enters the domain, since
packets carrying the proposed Source Address encoding are not
expected to cross the boundary.

## Return Reachability and ICMPv6 {#return-icmpv6}

Because the upper 104 bits of the outer IPv6 Source Address form a
valid and routable IPv6 prefix assigned to the ingress node,
return traffic and ICMPv6 messages addressed to any address within
this prefix are delivered to the ingress node according to normal
IPv6 forwarding within the limited domain.

To preserve return reachability for any value of the 24-bit service
identifier, the ingress node MUST treat all IPv6 addresses sharing
the upper 104-bit prefix as locally configured addresses, and MUST
accept and process IPv6 packets addressed to any of them. In other
words, the ingress node behaves, for the purpose of receiving
return traffic, as if the entire /104 prefix were assigned to it.
ICMPv6 error messages generated within the underlay, such as
Packet Too Big or Time Exceeded, that are addressed to the outer
IPv6 Source Address of an encapsulated packet are therefore
delivered to the ingress node regardless of the value of the lower
24 bits, and can be associated with the corresponding bridge
domain through the encoded service identifier.

The detailed handling of such messages by the ingress node,
including any subsequent action on the encapsulation or on the
inner-frame size, is implementation-specific and outside the scope
of this document.

## Path MTU Discovery

The Layer-2 encapsulation defined in this document adds an outer
IPv6 header and, optionally, a Segment Routing Header to the inner
Ethernet frame. As with any tunneling mechanism, this can cause
the encapsulated packet to exceed the Path MTU between the ingress
and the egress nodes within the limited domain.

When this happens, intermediate routers along the path generate
ICMPv6 Packet Too Big messages addressed to the outer IPv6 Source
Address of the encapsulated packet. As discussed in {{return-icmpv6}},
these messages are delivered to the ingress node regardless of
the value of the lower 24 bits of the Source Address, because the
ingress node treats all addresses sharing the allocated 104-bit
prefix as locally configured.

The detailed handling of received Packet Too Big messages by the
ingress node, including any association with the corresponding
bridge domain and any subsequent action on the
encapsulation or on the inner Ethernet frame size, is
implementation-specific and outside the scope of this document.
As a baseline, operators are expected to provision the underlay
MTU within the limited domain so that encapsulated packets, sized
to accommodate the maximum expected inner Ethernet frame plus the
outer IPv6 and SRH overhead, can traverse the domain without
fragmentation.

## Underlay ECMP and Entropy

The encoding defined in this document places the 24-bit Layer-2
service identifier in the lower 24 bits of the outer IPv6 Source
Address. Underlay routers performing equal-cost multipath (ECMP)
and load balancing on SRv6 traffic typically use the IPv6 Flow
Label of the outer IPv6 header to compute entropy, in line with
established SRv6 practice. Hardware fast-path implementations do
not generally include the lower bits of the Source Address in the
entropy hash.

As a consequence, the encoding defined in this document neither
improves nor degrades the underlay ECMP behavior. Operators that
require flow-level entropy across SRv6 Layer-2 tunnels SHOULD
populate the Flow Label of the outer IPv6 header at the ingress
node, independently of the encoded Layer-2 service identifier,
following normal SRv6 entropy practices.

## Operational Tooling

Within the limited domain, source-address-based filtering, policy
enforcement, accounting, and monitoring tools may be exposed to
outer IPv6 Source Addresses that share a common upper prefix and
differ in the lower 24 bits according to the encoded Layer-2
service identifier. Operational tools deployed within the domain
SHOULD be configured to interpret such addresses consistently with
the encoding defined in this document, so that packets belonging
to different bridge domains are not aggregated or
distinguished in unexpected ways.

# Security Considerations

The mechanism defined in this document uses the 24 least
significant bits of the outer IPv6 Source Address to identify the
bridge domain associated with a tunneled frame.

As a consequence, unauthorized modification of the outer IPv6
Source Address may cause a packet to be associated with the
wrong bridge domain at the decapsulating node, resulting in
traffic misdelivery between bridge domains.

For this reason, the mechanism defined in this document is intended
for deployment within a limited domain under a single
administrative authority, as discussed in {{applicability}}.
Within the limited domain, the operator is expected to protect the
integrity of the outer IPv6 Source Address against unauthorized
modification, in order to avoid traffic misdelivery between
bridge domains.

The security implications also depend on the control-plane
mechanism used to assign the 24-bit service identifier and to
configure the mapping between service identifiers and local
bridge domains. Protection of such control-plane
mechanisms is outside the scope of this document.

# IANA Considerations

This document proposes two new SRv6 behaviors, denoted as
`End.DT2U.SA` and `End.DT2M.SA`, which correspond, respectively,
to source-address-based service-identification variants of the
`End.DT2U` and `End.DT2M` behaviors defined in {{RFC8986}}.

If this document is progressed, IANA allocations will be needed
for both the `End.DT2U.SA` and the `End.DT2M.SA` behaviors in the
"SRv6 Endpoint Behaviors" registry.

This document does not request any additional IANA action in this
version.

# Implementation Status

This section is to be removed before publication.

A Linux prototype implementation of an SRv6 Layer-2 overlay
following the architectural model described in this document is
available and is described in {{HYDN-MAITI-2026}}. The prototype
includes a native bridge-integrated SRv6 Layer-2 tunnel netdevice
(sr6) and Linux support for the `End.DT2U` behavior of
{{RFC8986}}. The integration of the source-address-based
service-identification mechanism specified in this document, and
the corresponding support for the `End.DT2U.SA` and
`End.DT2M.SA` behaviors, is work in progress.

--- back

# Acknowledgments
{:numbered="false"}

TBD.
