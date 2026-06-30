---
docname: draft-westerlund-moq-overview-latest
title: "Media over QUIC Overview"
abbrev: MoQ Overview
obsoletes:
cat: info
ipr: trust200902
wg: moq
area: WIT
submissiontype: IETF

venue:
  group: Media over QUIC (MOQ)
  mail: moq@ietf.org
  github: https://github.com/gloinul/draft-westerlund-moq-overview

author:
-
   ins:  M. Westerlund
   name: Magnus Westerlund
   org: Ericsson
   email: magnus.westerlund@ericsson.com
-
   ins: Z. Sarker
   name: Zaheduzzaman Sarker
   org: Nokia
   email: zaheduzzaman.sarker@nokia.com

informative:

   RFC9000:
   RFC9605:
   RFC9420:
   RFC9750:
   I-D.ietf-moq-transport:
   I-D.ietf-moq-msf:
   I-D.ietf-moq-cmsf:
   I-D.ietf-moq-secure-objects:
   I-D.ietf-moq-privacy-pass-auth:
   I-D.ietf-moq-c4m:
   I-D.ietf-moq-loc:
   I-D.ietf-webtrans-overview:
   I-D.englishm-moq-relay-dos:

normative:


--- abstract

This document provides a high-level overview of the Media over QUIC
(MoQ) protocol suite. It describes the architecture, data model,
transport protocol, streaming formats, and security mechanisms that
together enable scalable low-latency media delivery over the Internet.
The document explains how these components relate to each other and
how they are composed to create interoperable media applications.

--- middle

# Introduction {#introduction}

Media over QUIC (MoQ) incorporates a set of
specifications for scalable, low-latency delivery over the Internet.
The core output is MOQT, a publish/subscribe transport protocol built
on QUIC {{RFC9000}} that uses intermediate relays to achieve
distribution at scale. On top of MOQT, additional specifications
define streaming formats, media containers, and end-to-end security
mechanisms.

This document provides a high-level overview of these component protocols, how
they relate to each other, and how they are composed to create
interoperable applications. It is intended as an entry point for
implementers, reviewers, and protocol designers who want to understand
the overall system before reading the individual normative
specifications.

Although the primary motivation of MoQ is media delivery,
MOQT itself is content-agnostic — it transports opaque objects
without knowledge of their payload. Media-specific logic (codecs,
containers, adaptive bitrate switching) is defined in separate
streaming format specifications that layer on top of the transport.
This separation means the protocol can serve other applications that
benefit from the same publish/subscribe, relay-based delivery model.

## Motivation {#motivation}

Today's media delivery landscape uses separate protocols for different
stages of the pipeline. Content is typically ingested using protocols
like RTMP, then repackaged at origin servers for distribution via
HTTP-based protocols such as HLS or DASH. This separation introduces
repackaging overhead, limits end-to-end latency, and creates
complexity at the boundaries between protocol domains.

MoQ addresses this by providing a single protocol that can carry media
from original publisher to end subscriber, including through
intermediate relay nodes. The design is driven by four goals:

Latency:
: TCP-based protocols suffer from head-of-line blocking and are slow
  to respond to congestion. MoQ leverages QUIC's multiplexed streams
  and partial reliability to minimise queuing latency and react
  quickly to changing network conditions.

Leveraging QUIC:
: QUIC provides parallel streams with independent loss recovery,
  unreliable datagrams, stream priorities, and flow control. MoQ
  maps its data model onto these features to achieve flexible
  trade-offs between latency and reliability.

Convergence:
: A single protocol from contribution through distribution eliminates
  the need for intermediate repackaging. The same MoQ objects
  produced by an encoder can be delivered to end subscribers without
  transformation at relay nodes.

Relays:
: Scalable media delivery requires intermediaries that can cache and
  fan out content. MoQ treats relays as first-class protocol
  participants: object metadata is structured so that relays can
  route, prioritise, and cache content without accessing the media
  payload itself.

The resulting protocol is a general-purpose pub/sub delivery layer.
The media-specific aspects — how audio and video samples are
packaged, how tracks are described in a catalog, how adaptive
bitrate switching works — are defined by separate streaming format
specifications that build on MOQT. Applications outside media
delivery can use MOQT directly with their own object formats,
without adopting any of the media-specific layers.

## Document Scope {#scope}

This document is informational. It does not define protocol behaviour
or create new requirements. Its purpose is to:

- Orient readers to the MoQ architecture and its components.
- Explain how the components relate to each other.
- Point readers to the relevant normative specification for each
  topic.

The covered specifications are:

- MOQT (Media over QUIC Transport) {{I-D.ietf-moq-transport}} —
  the publish/subscribe transport protocol.
- MSF (MOQT Streaming Format) {{I-D.ietf-moq-msf}} — a streaming
  format defining media packaging, catalog, and timeline.
- CMSF (CMAF-compliant MSF) {{I-D.ietf-moq-cmsf}} — an extension
  of MSF for CMAF/ISO-BMFF packaged media.
- MoQ Secure Objects {{I-D.ietf-moq-secure-objects}} — end-to-end
  authenticated encryption for media objects.
- Privacy Pass Authentication {{I-D.ietf-moq-privacy-pass-auth}} —
  privacy-preserving token-based authorization.
- Common Access Tokens for MoQ (C4M) {{I-D.ietf-moq-c4m}} —
  bearer token authorization scheme.
- LOC (Low Overhead Container) {{I-D.ietf-moq-loc}} — a minimal
  media container format.

## Terminology {#terminology}

This document uses the following terms. Definitions are consistent
with {{I-D.ietf-moq-transport}}.

Client:
: The party initiating a Transport Session.

Server:
: The party accepting an incoming Transport Session.

Publisher:
: An endpoint that handles subscriptions by sending objects from a
  track.

Subscriber:
: An endpoint that subscribes to and receives tracks.

Original Publisher:
: The initial publisher of a given track — the entity that creates
  the media content.

End Subscriber:
: A subscriber that consumes the content and does not forward it to
  other subscribers.

Relay:
: An entity that is both a publisher and a subscriber, forwarding
  content between other endpoints. Relays are neither the original
  publisher nor the end subscriber.

Track:
: A time-ordered sequence of groups, identified by a Full Track
  Name. A track is the unit of subscription.

Group:
: A collection of objects within a track that forms a join point.
  Groups are independently useful — objects within a group should
  not depend on objects in other groups.

Object:
: The smallest addressable data unit in MoQ. An object is an
  immutable sequence of bytes identified by its track, group ID,
  and object ID.

Subgroup:
: A sequence of objects within a group that share a QUIC stream.
  Objects in a subgroup have consistent priority and dependency
  relationships.

Transport Session:
: A QUIC connection or WebTransport session over which MOQT
  operates.


# Use Cases {#use-cases}

As noted in {{introduction}}, MOQT is a content-agnostic
publish/subscribe protocol. It can carry any data that fits the
track/group/object model — media is simply the use case that
motivated its design. Other potential applications include IoT
telemetry distribution, collaborative document state
synchronisation, real-time game state replication, and financial
market data feeds.

This section describes the primary media use cases that have driven
the protocol design. They illustrate the requirements that shaped
the architecture, but do not represent an exhaustive list of what
MOQT can be used for.

## Live Streaming Media Delivery {#uc-live}

In live streaming, an original publisher encodes and publishes media
content that is delivered through one or more relays to a potentially
large number of end subscribers.

~~~
 Original                                         End
 Publisher --> Relay --> Relay --> ... --> Relay --> Subscribers
                                                   (many)
~~~

Key characteristics:

- One-to-many distribution, typically through a tree of relays.
- Target end-to-end latency ranges from under one second to several
  seconds, depending on the application.
- Subscribers may join at any time and must be able to start playback
  from a recent group boundary.
- Adaptive bitrate (ABR) switching between alternate encodings of the
  same content allows subscribers to adapt to varying network
  conditions.
- The original publisher produces a catalog describing available
  tracks (resolutions, bitrates, codecs) so subscribers can make
  informed selections.

## Real-Time Conferencing {#uc-conferencing}

In conferencing, multiple participants each act as both original
publisher and end subscriber. Each participant publishes their own
audio and video tracks and subscribes to tracks from other
participants.

~~~
 Participant A ---+            +--- Participant C
                  |            |
                  v            v
              +-------+    +-------+
              | Relay |<-->| Relay |
              +-------+    +-------+
                  ^            ^
                  |            |
 Participant B ---+            +--- Participant D
~~~

Key characteristics:

- Many-to-many communication with low participant counts (tens to
  hundreds).
- Target latency is interactive: below 500 milliseconds end-to-end.
- Each participant subscribes to a subset of available tracks (e.g.,
  active speakers only).
- Scalable video coding (SVC) or simulcast may be used, with the
  relay or subscriber selecting appropriate layers.
- End-to-end encryption is particularly important since relays
  should not have access to media content.

## Content Contribution and Primary Distribution {#uc-ingest}

In content contribution (ingest), a professional encoder at a
production site publishes high-quality media to a relay operated by a
distribution service (e.g., a CDN).

~~~
 Encoder/       CDN Ingress      CDN Edge
 Producer  -->  Relay       -->  Relay(s)  -->  Subscribers
~~~

Key characteristics:

- The ingress relay receives the contributed content and may
  transcode it into multiple representations (resolutions, bitrates).
- After transcoding, the distribution service becomes the original
  publisher of the new representations — the transcoded tracks are
  distinct from the contributed track.
- Reliability is prioritised over latency for the contribution link.
- The same MOQT protocol is used for both the contribution link
  (encoder to ingress) and the distribution links (edge relays to
  subscribers), achieving the convergence goal.


# Architecture and Nodes {#architecture}

## Roles {#roles}

A MOQT deployment involves endpoints in the following roles:

Publisher:
: Sends objects belonging to one or more tracks. A publisher
  responds to subscriptions and delivers matching objects to
  subscribers.

Subscriber:
: Requests tracks and receives objects. A subscriber issues
  SUBSCRIBE or FETCH requests to obtain content.

Original Publisher:
: The endpoint that creates and first publishes a track's content.
  The original publisher is responsible for the media encoding,
  object payload, and track properties.

End Subscriber:
: The final consumer of a track's content. An end subscriber
  decodes and renders the media.

A single endpoint may act in multiple roles simultaneously. In
conferencing, each participant is both an original publisher (for
their own media) and an end subscriber (for other participants'
media).

## Relays {#relays}

Relays are intermediary nodes that are both publishers (to downstream
subscribers) and subscribers (to upstream publishers). They terminate
MOQT sessions on each side independently — a relay is not a
transparent proxy.

Relays provide:

- Fan-out: A single upstream subscription serves multiple downstream
  subscribers requesting the same track.
- Caching: Objects can be stored and served to subscribers that join
  later, without fetching from the original publisher again.
- Geographic distribution: Relays can be deployed close to
  subscribers to reduce latency.

Relays are content-agnostic: they forward object payloads without
interpreting or modifying them. This enables end-to-end encryption
where relays cannot access media content, while still performing
their routing and caching functions based on object metadata
(track names, group IDs, priorities, properties).

Relays are enforcement points for access and authorization. In many
deployments that have any restriction on access of the content the
relay is expected to be the node that receives proof of authorization
and will be responsible to verify and periodically re-verify this
proof before delivering content.

Relays are also not necessarily application-agnostic when it
comes to deployment and operation. The configuration of relay
topologies, authorization policies, caching strategies, and resource
limits are application-specific concerns. A relay serving a live
streaming platform operates differently from one supporting a
conferencing service, even though the protocol mechanics are
identical. Moreover, eventhough relays need not parse media payload
formats in order to forward content, but they may still apply
application-aware policy based on visible metadata, naming
conventions, and authorization state.

For further discussion of relay operational considerations,
including denial-of-service resilience, see
{{I-D.englishm-moq-relay-dos}}.

## Deployment Topologies {#topologies}

MOQT supports several deployment topologies:

Point-to-point:
: A publisher connects directly to a subscriber with no
  intermediate relay. Suitable for simple scenarios or testing.

~~~
 Publisher <--> Subscriber
~~~

Single relay:
: A publisher connects to a relay, which serves one or more
  subscribers. The relay provides fan-out and caching.

~~~
 Publisher <--> Relay <--> Subscriber(s)
~~~

Relay tree:
: A publisher connects to an origin relay, which fans out to edge
  relays closer to subscribers. This is the typical CDN deployment
  model.

~~~
 Publisher <--> Origin Relay <--> Edge Relay(s) <--> Subscriber(s)
~~~

Relay mesh:
: Relays interconnect with multiple peers, subscribing and
  publishing bidirectionally between each other. Each relay both
  receives content from and forwards content to its neighbouring
  relays. This topology suits conferencing and multi-publisher
  scenarios where content flows in multiple directions
  simultaneously and there is no single origin.

~~~
 Publisher A <--> Relay 1 <----> Relay 2 <--> Subscriber C
                   ^    ^       ^    ^
                   |     \     /     |
                   |      v   v      |
                   |     Relay 3     |
                   |       ^         |
                   v       |         v
         Subscriber B  Publisher D  Subscriber E
~~~

In all topologies, each hop is an independent MOQT session. Relays
manage subscriptions on each session independently and can
aggregate multiple downstream subscriptions into a single upstream
subscription.

MOQT's session migration mechanism (GOAWAY) allows relays to be
restarted or replaced without disrupting end-to-end delivery:
subscribers re-establish sessions to a new relay and migrate their
subscriptions.


# Data Model {#data-model}

MOQT defines a hierarchical data model for organising media content.
Understanding this model is essential to working with any part of
the MoQ protocol suite. The data model is defined in Section 2 of
{{I-D.ietf-moq-transport}}.

## Hierarchy {#hierarchy}

The data model has four levels:

~~~
 Track
   └── Group (join point)
         └── Subgroup (stream-mapped unit)
               └── Object (smallest addressable payload)
~~~

Track:
: A named, time-ordered sequence of groups. Tracks are the unit of
  subscription — a subscriber requests a track by its Full Track
  Name and receives objects from that track.

Group:
: A temporal collection of objects within a track. A group
  represents a join point: a subscriber can begin receiving content
  at any group boundary without needing prior groups. For video,
  a group typically corresponds to a Group of Pictures (GOP)
  starting with a random access point.

Subgroup:
: A sequence of objects within a group that are sent on a single
  QUIC stream. Objects in a subgroup share priority and have
  dependency relationships consistent with in-order delivery.
  Subgroups enable the priority mechanism to favour certain objects
  (e.g., base-layer frames) over others within the same group.

Object:
: The basic data element — an addressable, immutable sequence of
  bytes. Each object is uniquely identified by its Full Track Name,
  group ID, and object ID. Objects within a group are ordered by
  object ID.

## Track Naming {#naming}

Every track is identified by a Full Track Name consisting of two
parts:

Track Namespace:
: An ordered tuple of one or more fields ( up to 32) (e.g.,
  "example.com", "meeting123", "alice"). The namespace provides
  hierarchical structure that relays use for prefix-based routing
  and subscription matching.

Track Name:
: A byte sequence identifying a specific track within the
  namespace (e.g., "video-hd", "audio", "catalog").

The combination of namespace and track name is unique within a
given scope (the set of servers sharing a common connection URI
authority and path). This uniqueness guarantee means that a Full
Track Name can serve as a cache key.

## Properties {#properties}

Tracks and objects can carry additional metadata called Properties.
Properties are visible to relays and can influence how content is
distributed.

Track Properties:
: Metadata associated with an entire track. Examples include
  default publisher priority, group order preference, maximum
  cache duration, and delivery timeouts. Track Properties are
  communicated in control messages (PUBLISH, SUBSCRIBE_OK,
  FETCH_OK).

Object Properties:
: Per-object metadata carried in object headers. MOQT defines
  the framework for object properties (examples include gap
  indicators - Prior Group ID Gap, Prior Object ID Gap, while
  media-specific specifications such as timestamps and frame
  markings are defined in LOC and MSF define additional semantics
  for properties relevant to encoded audio/video samples.

Immutable Properties:
: A special container within either track or object properties whose
  contents must not be modified or removed by relays. Properties
  marked as Immutable so they can be end-to-end authenticated and thus
  verified as not being changed or removed by any relay.

Properties are registered in IANA registries and use a Key-Value-Pair
encoding. Relays that do not understand a property must forward it
unchanged.

## Object Immutability and Cacheability {#immutability}

A fundamental property of MOQT objects is immutability: once an
object is published, its payload must never change. Two objects with
the same Full Track Name, group ID, and object ID must contain
identical bytes.

This guarantee enables:

- Caching at relays without cache-invalidation complexity.
- Deduplication when the same object arrives from multiple upstream
  sources.
- End-to-end integrity verification — a subscriber can detect if an
  object has been tampered with.

An object can transition from "exists" to "does not exist" (e.g.,
cache expiry), but its contents cannot be modified. The protocol
signals object status (normal, end-of-group, end-of-track) to
communicate lifecycle information.


# MOQT: The Transport Protocol {#transport}

Media over QUIC Transport (MOQT) {{I-D.ietf-moq-transport}} is the
publish/subscribe protocol that carries objects between publishers and
subscribers. MOQT treats all object payloads as opaque byte sequences
— it provides delivery, prioritisation, and caching without any
knowledge of the content being carried. This section provides an
overview of its key mechanisms.

## Session Establishment {#session-establishment}

MOQT runs over either native QUIC or WebTransport:

- Native QUIC: The client connects using ALPN "moqt" and identifies
  the server using a "moqt://" URI. The authority and path are
  communicated via Setup Options.
- WebTransport: The client establishes a WebTransport session over
  HTTP/3, using an HTTPS URI derived from the moqt:// URI.

MOQT can run over native QUIC or over WebTransport over HTTP/3.
While both provide stream multiplexing and secure transport, the
deployment model and some capabilities are constrained differently,
especially in browser-based WebTransport environments. Applications
should treat them as functionally similar substrates, not strictly
identical ones.

After the transport connection is established, each endpoint opens a
unidirectional control stream and sends a SETUP message. The SETUP
exchange negotiates the protocol version and extensions (Setup
Options) before any other messages are exchanged.

## Publishing and Subscribing {#pub-sub}

MOQT provides several mechanisms for publishers and subscribers to
exchange content:

SUBSCRIBE:
: A subscriber requests newly published objects for a track. The
  publisher responds with SUBSCRIBE_OK and begins forwarding
  matching objects as they are produced. Subscription filters
  (Largest Object, Next Group, AbsoluteStart, AbsoluteRange)
  control which objects are delivered.

PUBLISH:
: A publisher initiates a subscription by pushing content to a
  subscriber. This inverts the usual direction — the publisher
  offers a track rather than waiting for a subscription request.

FETCH:
: A subscriber requests a specific range of previously published
  objects. Unlike SUBSCRIBE, which delivers future objects, FETCH
  retrieves historical data. A Joining Fetch can be associated with
  a subscription to fill the gap between stored history and the
  live edge.

Namespace Discovery:
: Publishers advertise available namespaces via PUBLISH_NAMESPACE.
  Subscribers discover content by sending SUBSCRIBE_NAMESPACE
  (to learn about matching namespaces) or SUBSCRIBE_TRACKS (to
  receive PUBLISH messages for tracks within matching namespaces).
  Relays use namespace prefix matching to route subscriptions to
  the appropriate upstream publishers.

All subscription interactions follow a state machine: Idle →
Pending → Established → Terminated. Either endpoint can terminate
a subscription — the subscriber via cancellation, the publisher
via PUBLISH_DONE.

## Priorities and Delivery {#priorities}

When network capacity is insufficient to deliver all subscribed
content, MOQT's priority system determines what gets delivered
first:

- Subscriber Priority: Set per-subscription by the subscriber to
  express relative importance among its own subscriptions.
- Publisher Priority: Set per-object by the publisher to express
  relative importance within a track (e.g., base layer vs.
  enhancement layer).
- Group Order: Ascending (oldest first) or descending (newest
  first) delivery of groups within a subscription.

Under congestion, lower-priority content may be delayed or dropped
entirely. Two timeout mechanisms support controlled data loss:

- Object Delivery Timeout: If an object cannot be sent within this
  duration after its first byte was produced, it is discarded.
- Subgroup Delivery Timeout: If a completed subgroup's data is not
  fully acknowledged within this duration, the stream is reset.

Objects can be sent on QUIC streams (reliable, in-order within the
stream) or as QUIC datagrams (unreliable, low-latency). The choice
is the Object Forwarding Preference, set per-object by the original
publisher.

## Extensibility and Session Management {#extensibility}

MOQT is designed for evolution without breaking existing
implementations:

Setup Options:
: Negotiated per-session in the SETUP exchange. Unknown options are
  ignored, allowing new features to be introduced without breaking
  older endpoints.

Message Parameters:
: Per-message metadata included in control messages. Parameters are
  peer-to-peer and are not forwarded by relays.

Properties:
: Per-track and per-object metadata that IS forwarded and cached by
  relays. Properties use IANA-registered type codes. Unknown
  properties must be forwarded unchanged.

GREASE:
: Reserved code points in all registries ensure implementations
  correctly handle unknown values without closing sessions.

Session migration is supported via the GOAWAY message. A server
sends GOAWAY to signal that the client should establish a new session
(optionally to a different URI) and migrate its subscriptions. This
enables server maintenance and load balancing without disrupting
media delivery.


# Streaming Formats {#streaming-formats}

MOQT is content-agnostic — it transports opaque object payloads
without interpreting them. A streaming format defines how media
content is packaged into MOQT objects and how subscribers discover
and select tracks. Streaming formats bridge the gap between raw
media encodings and the MOQT data model.

## Role of a Streaming Format {#sf-role}

A streaming format specifies:

- How encoded media samples are mapped to MOQT objects and groups.
- A catalog format describing available tracks and their properties.
- Rules for time-alignment and ABR switching between tracks.
- Timeline information mapping MOQT group/object IDs to media
  presentation time.
- Any additional metadata tracks (events, logs, metrics).

Without a streaming format, a subscriber receiving objects would have
no way to discover what content is available, how to decode it, or
how to synchronise multiple tracks.

## MSF: MOQT Streaming Format {#msf}

The MOQT Streaming Format (MSF) {{I-D.ietf-moq-msf}} is the primary
streaming format defined for MoQ. It uses LOC (Low
Overhead Container) {{I-D.ietf-moq-loc}} packaging, where each
encoded media sample (audio frame or video frame) is placed in a
separate MOQT object.

MSF defines:

Catalog:
: A JSON track named "catalog" that describes all available tracks
  in a namespace. The catalog includes codec information, bitrates,
  resolutions, language, and relationships between tracks (render
  groups for synchronised playback, alternate groups for ABR
  switching). Delta updates allow efficient signalling of track
  additions and removals.

Media Timeline:
: An optional track that maps MOQT group/object IDs to media
  presentation timestamps and wallclock times. This enables seeking
  and synchronisation. A template mechanism supports regular
  patterns without requiring an explicit timeline track.

Event Timelines:
: Optional metadata tracks carrying time-synchronised application
  data (e.g., sports scores, ad insertion markers, GPS coordinates).

Publish Tracks:
: Tracks defined in the catalog to which the subscriber can publish
  data back (e.g., QoE metrics, logs).

MSF supports content with target latencies ranging from real-time
(under 500ms) to standard live (several seconds) to video-on-demand.

## CMSF: CMAF-Compliant Streaming Format {#cmsf}

CMSF {{I-D.ietf-moq-cmsf}} extends MSF to support CMAF (Common
Media Application Format) packaged media. It is designed for
workflows that use ISO-BMFF containers and existing DRM systems.

Key differences from base MSF:

- Objects contain CMAF Chunks (moof+mdat boxes) rather than raw
  codec samples.
- Initialization segments (CMAF Headers) are carried in the catalog
  as base64-encoded inline data.
- Content protection uses Common Encryption (CENC) with commercial
  DRM systems (Widevine, PlayReady, FairPlay) rather than the
  application-layer encryption defined by MoQ Secure Objects.
- SAP-type event timelines signal stream access points for seeking.

CMSF targets deployments that need interoperability with existing
DASH/HLS ecosystems and hardware-based content decryption modules.

## Layering {#layering}

The relationship between the components is:

~~~
 +------------------+
 | Application      |
 +------------------+
 | Streaming Format |  <-- MSF or CMSF
 | (catalog, rules) |
 +------------------+
 | Media Container  |  <-- LOC or CMAF
 +------------------+
 | MOQT             |  <-- transport protocol
 +------------------+
 | QUIC / WebTrans  |
 +------------------+
 | UDP / IP         |
 +------------------+
~~~

The streaming format layer defines the media semantics. The
container layer defines how individual samples are serialised.
MOQT provides the delivery machinery. Applications select the
appropriate combination for their needs.


# Security Architecture {#security}

MoQ employs a layered security model. Each layer addresses different
threats and operates at a different scope.

## Hop-by-Hop Security {#hbh-security}

MOQT runs over QUIC or WebTransport, both of which use TLS 1.3 for
the underlying connection. This provides:

- Confidentiality and integrity of all data on each hop.
- Server authentication (and optionally mutual authentication).
- Protection against on-path attackers between adjacent nodes.

However, hop-by-hop security does NOT protect content from relays.
A relay terminates the TLS session and has access to all MOQT
message contents, including object payloads, unless additional
end-to-end protection is applied.

## End-to-End Object Encryption {#e2e-security}

MoQ Secure Objects {{I-D.ietf-moq-secure-objects}} provides
end-to-end authenticated encryption of media object payloads. It is
based on the SFrame mechanism ({{RFC9605}}).

The scheme:

- Encrypts the object payload and optional encrypted properties.
- Authenticates the group ID, object ID, Full Track Name, and
  immutable properties.
- Derives per-track keys from a shared base key using HKDF.
- Forms nonces from the group ID and object ID, guaranteeing
  uniqueness.

Relays can still route and cache objects based on unencrypted
metadata (track names, group IDs, priorities, object properties)
but cannot access the media content. The architecture separates
metadata required for routing/caching from metadata that can be
authenticated or encrypted end-to-end. The exact protected
elements depend on the object security scheme and application
profile.

Key distribution is out of scope for the MoQ specifications —
applications use external key management systems (e.g., MLS-based key
distribution for conferencing {{RFC9750}},{{RFC9420}}, or out-of-band provisioning
for streaming).

## Authorization {#authorization}

In general, MOQT defines where authorization material can be
carried in protocol exchanges, while the semantic interpretation
and verification procedure depend on the selected authorization
scheme and the relay or publisher’s local policy.

MOQT provides a generic AUTHORIZATION TOKEN parameter that can be
included in control messages (SUBSCRIBE, FETCH, PUBLISH,
PUBLISH_NAMESPACE, and others). Both publishers and subscribers use
this single mechanism to present credentials; relays verify the
token before forwarding content or establishing upstream
subscriptions. Tokens can be registered with session-scoped aliases
to avoid retransmitting large values on every message.

The two defined authorization schemes that operate
over this mechanism:

- Privacy Pass Authentication {{I-D.ietf-moq-privacy-pass-auth}} —
  provides privacy-preserving, unlinkable authorization using
  Privacy Pass tokens. It supports fine-grained access control
  through namespace and track name matching rules, and defines
  mechanisms for continuous re-authorization over long-lived
  sessions using batched tokens or a reverse issuance flow.

- Common Access Tokens (C4M) {{I-D.ietf-moq-c4m}} — provides a
  bearer token scheme using structured tokens with claims that
  express authorization scope. C4M follows a model familiar from
  HTTP-based systems.

The choice between schemes depends on application requirements:
Privacy Pass prioritises subscriber anonymity, while C4M offers
simpler integration for deployments with existing identity
infrastructure.

The streaming format catalog can signal authorization requirements
per-track (via the authInfo field in MSF), allowing subscribers to
discover what credentials they need before attempting to subscribe.

## Privacy Considerations {#privacy}

MoQ's relay-based architecture means that intermediaries necessarily
observe certain metadata in the course of routing and caching
content. Key privacy considerations include:

- Relay visibility of metadata: Track namespaces, track names,
  group IDs, and object properties are visible to every relay in
  the delivery path. This metadata can reveal information about
  content and participants even when payloads are end-to-end
  encrypted.

- Traffic analysis: Object sizes, timing patterns, and
  subscription behaviour can allow relays or on-path observers to
  infer what content is being consumed or who is communicating.

- Subscriber unlinkability: Without privacy-preserving
  authorization (such as Privacy Pass), relays can correlate a
  subscriber's identity across multiple sessions or subscriptions.

MOQT provides mechanisms that applications can use to mitigate
these risks, including padding (to obscure traffic patterns) and
token schemes that support unlinkable authentication. However,
achieving strong privacy requires deliberate application design —
for example, using generic namespace structures and consistent
object sizes.


# Defining a MoQ Application {#application}

MOQT, streaming formats, container formats, and security mechanisms
are composable building blocks. A complete MoQ application is defined
by specifying which components to use and how they are configured.
This set of choices is also what creates an interoperability point:
two implementations that make the same choices can exchange content
without additional negotiation.

## Application Design Choices {#design-choices}

Defining a MoQ application requires decisions in each of the
following areas:

Transport version:
: Which MOQT version (ALPN value) the application requires. This
  determines the available protocol features.

Streaming format and catalog:
: How media or data is packaged into MOQT objects, and how
  available tracks are described to subscribers. An application may
  adopt an existing format (MSF, CMSF) or define a new one.

Container and codecs:
: What container format wraps the encoded samples (e.g., LOC, CMAF)
  and which codecs are supported.

Security:
: Whether end-to-end encryption is required (and which scheme), and
  which authorization scheme(s) are used to control access.

Relay deployment model:
: How relays are deployed and operated — tree, mesh, or hybrid —
  and what trust and authorization policies apply between nodes.

Extensions:
: Any mandatory MOQT extensions (Properties, Setup Options) that
  endpoints must support. Applications that need relay-visible
  metadata beyond what MOQT defines can register new Properties or
  use the application-reserved type ranges.

If existing components do not cover an application's needs, the MoQ
framework supports extension without modifying the base protocol:
new streaming formats can define their own catalog structures and
packaging rules, new Properties can be registered via IANA, and new
event timeline types can be defined for synchronised metadata.

## Existing Application Definitions {#existing-apps}

The MoQ working group has defined two complete application
specifications that illustrate the design choices above:

MSF (MOQT Streaming Format):
: Uses MOQT + LOC packaging + a JSON catalog. Targets real-time and
  live streaming with LOC's minimal per-sample overhead. Supports
  ABR switching via time-aligned alternate groups, media timelines,
  event timelines, and publish tracks for subscriber-to-platform
  communication. Defined in {{I-D.ietf-moq-msf}}.

CMSF (CMAF-compliant Streaming Format):
: Extends MSF with CMAF/ISO-BMFF packaging and Common Encryption
  (CENC) for DRM integration. Targets deployments that interoperate
  with existing DASH/HLS ecosystems and hardware content decryption.
  Defined in {{I-D.ietf-moq-cmsf}}.

Both specifications make concrete choices for every area listed in
{{design-choices}}, producing fully interoperable application
definitions.

## Reading Guide {#reading-guide}

The following table maps common goals to the relevant specifications:

| Goal | Specification |
|------|---------------|
| Understand the wire protocol and session mechanics | {{I-D.ietf-moq-transport}} |
| Build a live streaming or conferencing application using LOC | {{I-D.ietf-moq-msf}} |
| Use CMAF packaging with DRM | {{I-D.ietf-moq-cmsf}} |
| Add end-to-end encryption | {{I-D.ietf-moq-secure-objects}} |
| Implement privacy-preserving authorization | {{I-D.ietf-moq-privacy-pass-auth}} |
| Implement bearer-token authorization | {{I-D.ietf-moq-c4m}} |
| Understand the LOC container format | {{I-D.ietf-moq-loc}} |
| Operate relays and understand DoS resilience | {{I-D.englishm-moq-relay-dos}} |


# Security Considerations {#security-considerations}

The MoQ security model is described in {{security}}. The key
residual risks are:

Relay Trust:
: Relays see all MOQT metadata and, without E2E encryption, all
  object payloads. Deployments must carefully consider which
  entities operate relays and what authorization they require.

Cache Poisoning:
: If a relay accepts objects from an unauthorized publisher, it may
  serve corrupted content to subscribers. Relay implementations
  must verify publisher identity and authorization before caching.

Traffic Analysis:
: Even with E2E encryption, object sizes, timing patterns, and
  track names can reveal content characteristics. Padding and
  generic naming mitigate but do not eliminate this risk.

Authorization Token Replay:
: Token replay protection is delegated to the specific token
  scheme. Implementations must select schemes with appropriate
  replay resistance for their threat model.

Amplification:
: A single SUBSCRIBE can trigger unbounded data flow from a
  publisher or relay. Rate limiting, per-session resource caps,
  and the EXCESSIVE_LOAD error code provide defences. See
  {{I-D.englishm-moq-relay-dos}} for detailed analysis.


# IANA Considerations {#iana}

This document has no IANA actions.


# Contributors

The following persons have contributed to this document:

- Gwendal Simon

# Acknowledgments


--- back

