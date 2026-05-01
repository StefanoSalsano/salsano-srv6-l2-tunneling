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
tunnel-endpoint identification and Layer-2 service identification
with a compact and explicit identifier for each Layer-2 service
instance. VXLAN {{RFC7348}} is the most widespread example of this
operational model: the outer IP addressing identifies the remote
tunnel endpoint, while a 24-bit VXLAN Network Identifier (VNI),
carried in a dedicated header field outside the outer IP addressing,
identifies the specific Layer-2 service instance. This separation
has made VXLAN the de facto reference for scalable Ethernet
overlays.

SRv6 already defines Layer-2 endpoint behaviors in {{RFC8986}},
including `End.DT2U` for delivery of decapsulated Ethernet frames
into a local Ethernet domain through Layer-2 table lookup. These
behaviors provide the architectural basis for SRv6 Layer-2 overlays.
However, in current SRv6 practice the identification of the specific
Layer-2 service instance is typically absorbed into the outer IPv6
Destination Address, for example through a service uSID at the
egress node, so that endpoint identification, behavior selection,
and service-instance identification share the same address.

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

   VXLAN VNI (24b)        (service identifier    outer IPv6 SA
   -> L2 service          absorbed into the      lower 24 bits
      instance            Destination Address)   -> L2 service
                                                    instance
~~~
{: #fig-overview title="Service identification: VXLAN, current SRv6, and this document."}

Implementation aspects of an SRv6 Layer-2 overlay using this
service-identification model, including integration with the local
Layer-2 data plane, are discussed in {{HYDN-MAITI-2026}}.

# Problem Statement and Design Goals

SRv6 Layer-2 tunneling requires two distinct functions. First, a packet
must be delivered to the remote node that performs the decapsulation
behavior. Second, the decapsulating node must identify the specific
Layer-2 service, bridge domain, or virtual network to which the inner
frame belongs.

In VXLAN, these two functions are clearly separated. The outer IP
address identifies the remote tunnel endpoint, while the VXLAN Network
Identifier (VNI) identifies the Layer-2 service. The VNI is carried in a
dedicated 24-bit field and does not consume addressing space used for
tunnel delivery.

In SRv6, the outer IPv6 Destination Address and, more generally, the SID
list are naturally used for endpoint identification and behavior
selection. This is appropriate for steering packets to the decapsulating
node and invoking the desired SRv6 Layer-2 behavior. However, this does
not by itself provide an efficient way to identify the specific
Layer-2 service carried by the tunneled frame.

One possible solution is to encode the service identifier in the SRv6
Destination Address. This approach is unattractive because it consumes
bits that are valuable for locator encoding, endpoint identification,
behavior selection, and path steering. The problem is particularly acute
in deployments based on compressed SID representations such as uSID,
where the available SID space after the locator is limited. In such
environments, dedicating 24 bits to a VXLAN-like service identifier
significantly reduces the usable space for SRv6 forwarding semantics.

As a result, SRv6 Layer-2 tunneling lacks a compact and implementation-
friendly service identification mechanism with feature parity to VXLAN.
A practical solution should therefore satisfy the following design goals:

* preserve the Destination Address primarily for SRv6 endpoint
  identification and behavior selection;
* provide a compact service identifier comparable in size and role to
  the VXLAN VNI;
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

In VXLAN, the outer IP header identifies the remote tunnel endpoint,
while the VXLAN header carries a 24-bit VXLAN Network Identifier (VNI).
The VNI identifies the specific Layer-2 service, such as a bridge
domain, virtual network, or Layer-2 overlay instance.

This separation is simple and effective. The outer IP addressing is used
to reach the decapsulating node, while the VNI identifies the service
context in which the inner Ethernet frame is to be processed. As a
result, VXLAN provides a compact and explicit service identifier that is
independent of the outer IP addressing.

## Tunnel Identification in Current SRv6

In current SRv6 practice, especially when using compressed SID
representations such as uSID, the decapsulating tunnel endpoint is
typically identified by a local service uSID.

A uSID list commonly ends with:

* a locator uSID, which identifies the egress or decapsulation node; and
* one or more service uSIDs, which identify the behavior to be executed
  and the specific service instance at that node.

For example, a service uSID may identify:

* a specific VRF in a decapsulation behavior such as DTx;
* a specific Layer-2 tunnel context;
* a specific routing adjacency; or
* another local service instance bound to the node.

In this model, the IPv6 Destination Address is used not only to steer
the packet to the correct node, but also to identify the local service
instance to be applied at the endpoint.

This approach is workable, but it has an important limitation. With a
typical 2-octet uSID granularity, the service-uSID space available at a
node is on the order of 2^16 values. This space must be shared among all
local service instances of that node, cumulatively including VRFs,
Layer-2 tunnels, routing adjacencies, and other local behaviors.

As a consequence, the same limited service-uSID space is used both to
identify the behavior and to distinguish among all concrete service
instances supported by the node. In particular, allocating a VXLAN-like
24-bit Layer-2 service identifier directly in the SRv6 Destination
Address is impractical in uSID-based deployments, because it would
consume a substantial fraction of the available SID space.

This is a key difference from VXLAN. In VXLAN, the service identifier is
carried in a dedicated field outside the outer IP addressing. In current
SRv6 practice, service-instance identification is typically absorbed
into the Destination Address semantics, which makes scalable Layer-2
tunnel identification more difficult.

# Source-Address-Based Service Identification

This document proposes to encode the Layer-2 service identifier in the
24 least significant bits of the outer IPv6 Source Address.

The key idea is to separate the two functions that, in current SRv6
practice, are both absorbed into the Destination Address semantics:

* identification of the remote decapsulation node and of the SRv6
  behavior to be executed; and
* identification of the specific Layer-2 service instance associated
  with the tunneled frame.

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

The 24-bit service identifier carried in the Source Address may be used
to identify, for example, a bridge domain, a Layer-2 overlay instance,
or another local tunnel context associated with Layer-2 forwarding.

This document does not mandate a specific control-plane signaling
mechanism for the 24-bit service identifier. Such mechanisms are outside
the scope of this document and may be defined by future specifications.

The proposed use of the Source Address does not alter the role of the
Destination Address in SRv6 forwarding. Instead, it complements it by
providing a separate field for compact Layer-2 service identification.

# Relation to Existing SRv6 Behaviors

The mechanism proposed in this document is intended to define a new
SRv6 Layer-2 decapsulation behavior, denoted as `End.DT2U.SA`, rather
than to redefine the existing Layer-2 endpoint behaviors of
{{RFC8986}}.

Among the Layer-2 behaviors defined in {{RFC8986}}, `End.DT2U`
provides the closest functional reference for VXLAN-like Ethernet
overlays. `End.DT2U` performs decapsulation of an SRv6 packet
carrying an inner Ethernet frame and delivers the frame into a
local Ethernet domain through Layer-2 table lookup. In current
SRv6 practice, the specific Layer-2 service instance associated
with the decapsulating node is typically identified through the
Destination Address semantics, for example by using a service uSID.

The `End.DT2U.SA` behavior follows the same overall model of
Layer-2 decapsulation and delivery into a local Ethernet domain,
but introduces an additional service-identification function. The
egress node and the SRv6 behavior are identified through the outer
IPv6 Destination Address, while the specific Layer-2 service
instance is identified by the 24-bit value carried in the least
significant bits of the outer IPv6 Source Address, as defined in
this document.

For this reason, `End.DT2U.SA` can be viewed as an enhanced variant
of `End.DT2U`, in which the Layer-2 service identifier is carried
separately from the Destination Address. This provides a clearer
separation between endpoint identification and service-instance
identification, and avoids consuming Destination Address SID space
for VXLAN-like service identification.

The Layer-2 cross-connect behavior `End.DX2` of {{RFC8986}}, which
forwards the decapsulated frame toward a specific outgoing Layer-2
interface, is functionally distinct and is not the primary reference
for the mechanism defined in this document. Variants of the Layer-2
delivery behaviors for multicast or other forwarding semantics may
be defined in future specifications using a similar source-address
based service-identification model.

This document therefore assumes that the proposed mechanism is
specified as a distinct SRv6 behavior, namely `End.DT2U.SA`, rather
than as a backward-compatible reinterpretation of an existing
Layer-2 behavior.

# Encapsulation Procedure

At the ingress node, the encapsulating tunnel endpoint receives an
Ethernet frame that must be transported over an SRv6 Layer-2 tunnel.

The ingress node MUST construct an outer IPv6 header whose Destination
Address identifies the remote decapsulation node and the SRv6 behavior
to be executed at that node. In the approach defined by this document,
the behavior is `End.DT2U.SA`.

The ingress node MUST also assign a 24-bit service identifier to the
Layer-2 service instance associated with the frame. This identifier is
encoded in the 24 least significant bits of the outer IPv6 Source
Address, as defined in this document.

More precisely, let `SA` denote the outer IPv6 Source Address. The
ingress node MUST set:

~~~
SERVICE_ID = SA[23:0]
~~~

The remaining upper 104 bits of the outer IPv6 Source Address,
i.e. `SA[127:24]`, MUST preserve normal IPv6 Source Address semantics.
In particular, they MUST be assigned so that the source address remains
meaningful and reachable in the IPv6 domain where the tunnel is
deployed.

The resulting encapsulated packet consists of:

* an outer IPv6 header;
* optionally, an SRH if needed by the SRv6 policy;
* the original Ethernet frame as inner payload.

The ingress node MUST NOT modify the inner Ethernet frame except as
required by normal tunnel processing.

Operationally, the ingress node performs the following steps:

1. determine the remote decapsulation endpoint and the corresponding
   `End.DT2U.SA` service SID;
2. determine the 24-bit service identifier associated with the Layer-2
   service instance;
3. construct the outer IPv6 Source Address so that:
   * `SA[23:0]` carries the service identifier; and
   * `SA[127:24]` preserves valid IPv6 Source Address semantics;
4. encapsulate the Ethernet frame in the SRv6 packet;
5. forward the packet according to normal SRv6 processing.

The mapping between a Layer-2 service instance and the corresponding
24-bit service identifier is outside the scope of this document. It may
be statically provisioned or distributed by a control-plane mechanism.

# Decapsulation Procedure

When an encapsulated packet reaches the remote endpoint, SRv6
processing identifies the local behavior to be executed from the outer
IPv6 Destination Address and, if present, from the active SID in the
SRH. If the selected behavior is `End.DT2U.SA`, the node performs the
Layer-2 decapsulation procedure defined in this section.

The egress node MUST extract the 24 least significant bits of the outer
IPv6 Source Address and interpret them as the service identifier:

~~~
SERVICE_ID = SA[23:0]
~~~

The egress node MUST use this 24-bit value to identify the specific
Layer-2 service instance associated with the packet. The exact use of
this identifier is deployment-specific, but it is expected to identify,
for example, a bridge domain, a Layer-2 tunnel instance, or another
local Layer-2 forwarding context.

After identifying the service instance, the egress node removes the
outer SRv6 encapsulation and forwards the decapsulated Ethernet frame
according to the local Layer-2 forwarding context selected by the
service identifier.

Operationally, the egress node performs the following steps:

1. receive the SRv6 packet and identify the `End.DT2U.SA` behavior;
2. extract the 24-bit service identifier from the least significant
   bits of the outer IPv6 Source Address;
3. map the extracted service identifier to a local Layer-2 service
   instance;
4. remove the outer SRv6 encapsulation;
5. deliver the decapsulated Ethernet frame to the outgoing Layer-2
   context associated with the identified service instance.

If the extracted service identifier does not correspond to a valid
local Layer-2 service instance, the packet MUST be discarded.

The detailed error handling, OAM behavior, and optional ICMP reporting
for this case are left for future versions of this document.


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
to convey service or context information. The IPv6 VPN Service
Destination Option {{I-D.ietf-6man-vpn-dest-opt}} encodes VPN
identification in an IPv6 Destination Option, that is, in an
extension header rather than in an address field.
Application-aware IPv6 Networking
{{I-D.li-6man-apn-ipv6-encap}} similarly relies on extension-header
encapsulation to carry application-aware attributes. SRv6 Services
{{RFC9252}} and the SRv6 network programming model {{RFC8986}}
encode service-instance information in the Destination Address
through the SID structure, possibly in compressed form
{{RFC9800}}.

These approaches differ from the mechanism defined in this document
in two main ways. First, they either consume Destination Address
space or require the addition of an IPv6 extension header to carry
the service identifier, whereas the mechanism defined here reuses
existing space in the outer IPv6 Source Address. Second, they
target different service models, in particular IPv6 VPNs and
application-aware networking, whereas this document focuses on
Layer-2 service identification for SRv6 Layer-2 tunnels.

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

## Return Reachability and ICMPv6

Because the upper 104 bits of the outer IPv6 Source Address form a
valid and routable IPv6 prefix assigned to the ingress node,
return traffic and ICMPv6 messages addressed to the Source Address
are delivered to the ingress node according to normal IPv6
forwarding within the limited domain. This includes ICMPv6 error
messages generated within the underlay, such as Packet Too Big or
Time Exceeded, that are addressed to the outer IPv6 Source Address
of the original packet. The handling of such messages by the
ingress node is implementation-specific and outside the scope of
this document.

## Operational Tooling

Within the limited domain, source-address-based filtering, policy
enforcement, accounting, and monitoring tools may be exposed to
outer IPv6 Source Addresses that share a common upper prefix and
differ in the lower 24 bits according to the encoded Layer-2
service identifier. Operational tools deployed within the domain
SHOULD be configured to interpret such addresses consistently with
the encoding defined in this document, so that packets belonging
to different Layer-2 service instances are not aggregated or
distinguished in unexpected ways.

# Security Considerations

The mechanism defined in this document uses the 24 least significant
bits of the outer IPv6 Source Address to identify the Layer-2
service instance associated with a tunneled frame.

As a consequence, unauthorized modification of the outer IPv6
Source Address may cause a packet to be associated with the wrong
Layer-2 service instance at the decapsulating node. This can
result in traffic misdelivery across Layer-2 services or bridge
domains.

For this reason, the mechanism defined in this document is intended
for deployment within a limited domain under a single
administrative authority, as discussed in {{applicability}}.
Within the limited domain, the operator is expected to protect the
integrity of the outer IPv6 Source Address against unauthorized
modification, in order to avoid traffic misdelivery between
Layer-2 service instances.

The security implications also depend on the control-plane
mechanism used to assign the 24-bit service identifier and to
configure the mapping between service identifiers and local
Layer-2 service instances. Protection of such control-plane
mechanisms is outside the scope of this document.

# IANA Considerations

This document proposes a new SRv6 behavior, denoted as `End.DT2U.SA`.

If this document is progressed, an IANA allocation will be needed for
the `End.DT2U.SA` behavior in the relevant SRv6 behavior registry.

This document does not request any additional IANA action in this
version.

--- back

# Acknowledgments
{:numbered="false"}

TBD.
