---
docname: draft-westerlund-moq-overview-latest
title: "Media over QUIC Overview"
abbrev: MoQ Overview
obsoletes:
cat: std
ipr: trust200902
wg: moq
area: WIT
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"

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

informative:

   RFC9000:
   RFC9605:
   I-D.ietf-moq-transport:
   I-D.law-moq-warpstreamingformat:
   I-D.ietf-webtrans-overview:
   I-D.jennings-moq-secure-objects:
   I-D.mzanaty-moq-loc:

normative:


--- abstract

This document provides a high level overview of Media over QUIC
protocol components, how interoperability is specified for particular
use cases, and how things relate.


--- middle

# Introduction {#introduction}

Media over QUIC is an IETF WG and its output of set of specifications that
is inteded to be useable to address a couple of use cases realted to
low latency media delivery. This document provides a high level overview of
how these components relates to each other and what is needed for a specific
use case to define an interoperable media delivery solution.

The protocol work

## Use Cases

The development of the MoQ components considerd the following use
cases and what their main requirements. However, this may not prevent
the MoQ protocol components to be used for other use cases than what is
listed here.

In general the goal of the solution is enable ease of dynaimc scaling
of number of participanting endpoints and distributed cloud deployments.
The solution only assumes existance of unicast IP transport between involved
nodes.

### Live Streaming Media Delivery


### Real-time Conferencing


### Content Contribution




# Media Over QUIC overview

Meida over QUIC consists of several different components that allows
to build various different applications. Below we provide an overview
of these components after having discussed the involved type of nodes.


## Data Model





## Nodes

Consumer: An endpoint that want to consume (receive) one or more
       tracks from one or more track namespaces.

Publisher: An endpoint that can send a consumer one or more tracks for
       a given track namespace.

Original Publisher: The endpoint is the originator of one or more
       tracks for a particular track namespace.

Relay: An endpoint that is both a consumer and a publisher, but not an
       orignal publisher. Thus acting as relay between the orignal
       publisher and a consumer endpoint. Relays can also use another
       relay as publisher to enable scaling of the publisher nodes, as
       well as to handle geographical distribution of receivers to
       optimize performance.

## Media Over QUIC Transport

Media over QUIC Transprot (MOQT) {{I-D.ietf-moq-transport}} is the
protocol which is used between a publisher and a consumer. The
protocol supports a couple of different capabilities:

Subscribe: Request the publisher to forward media objects of a meida
       track as they are created or received from upstream publisher.

Fetch: Request the publisher to provide a specific range of media
       objects for a track that the publisher has previously received
       and cached.

Announce: A publisher announces a track namespace which is available
       for subscribe and fetch to the consumer.

When the consumer is subscribed to a track the publisher will try
forward all media objects of the track scheduling the transmission
according to priorities on the different objects competing for
available transport bitrates across all tracks the consumer is
subscribed to.

MOQT uses either QUIC {{RFC9000}} or WebTransport
{{I-D.ietf-webtrans-overview}} to transport its messages and media
objects. These transport protocols also are used to provide hop-by-hop
confidentiality and integrity.

MOQT is dependent on that there exists method for authorization and
authentication of the consumer and publisher to ensure that they have
the right to either access a track or publish a track in this track
namespace. This mechanism has not yet been defined.

The media objects related to a track are delivered over either
independent streams or using unrelaible datagrams depending on what
best suits the media and application when it comes to latency and
reliability.

Note: Add something about what happens when there are insufficinet
bandwidth.


## Catalog

The catalog is a media track description format that enables the
application to understand what track names exists in a track
namespace. The catalog describe for each track the track's properties
like peak bitrate, encoding format, relation to other tracks for lip
synch etc.

A catalog is not necessary for all applications, as the application
may have a fixed track name structure with known encodings and properties.

The WARP streaming format {{I-D.law-moq-warpstreamingformat}} defines
one catalog format that is specific for WARP. However, it is expected
that it can be the basis for other formats and be extended for other
applications. However, it is up to the application specifications to
define which catalog format they use.

## Media Formats

The media objects are opaque blobs from the perspective of
MOQT. However, the application need to decode these and thus they need
to be in a format supported by the consumer and its
application. Therefore, the application will need to define which
media formats that are required to be supported.

The MoQ WG is currently expected to define at least one media format,
namely the Low Overhead Media Container {{I-D.mzanaty-moq-loc}}. This
is a format that targets minimal overhead around the media objects
as defined by the WebCodecs browser API.

As CMAF have been dominant for a long time in general streaming
applications there has expessed some interest in defining a MoQ media
format using CMAF but currently no such proposal exists.

## End-to-End Object Security

MOQT provides only hop-by-hop confidenitality and integrity
protection. It has been expected that some applications want stronger
security properties, like end-to-end confidentiality as well as source
authentication. To meet these goals there exists a proposal for such a
mechanism. {{I-D.jennings-moq-secure-objects}} proposes that each
media object's payload is encapsualted in an SFRAME {{RFC9605}}.

Keying and cipher negotiation is currently application specific.

## Application Interopability

One or more application sharing the same needs for how Media over QUIC
operates can define a interoperaiblity point. An interoperability
point is defining which components and their versions that is need to
be supported to be interoperable.

The WARP Streaming Format {{I-D.law-moq-warpstreamingformat}} is such
an interoperability point that defines the following:

- That the media format is LOC and with additional requirements such
  that audio and video can be time aligned (lip synched).

- A Catalog format to describe the tracks.

- That MOQT as defined in {{I-D.ietf-moq-transport}} is used for Media transmission.






# Contributors

# Acknowledgments


--- back

