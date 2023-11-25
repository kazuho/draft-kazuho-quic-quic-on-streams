---
title: "QUIC Services for Streams"
docname: draft-kazuho-quic-quic-services-for-streams-latest
category: std
wg: QUIC
ipr: trust200902
keyword: internet-draft
stand_alone: yes
pi: [toc, sortrefs, symrefs]
author:
 -
    fullname: Kazuho Oku
    organization: Fastly
    email: kazuhooku@gmail.com

--- abstract

This document specifies a polyfill of QUIC version 1 that runs on top of
bi-directional streams such as TLS.


--- middle

# Introduction

QUIC version 1 ({{!RFC9000}}) is a bi-directional, authenticated transport-layer
protocol built on top of UDP ({{?RFC768}}). The protocol provides multiplexed
flow-controlled streams without head-of-line blocking as one of its core
services, along with low-latency connection establishment and efficient loss
recovery.

However, there are downsides with QUIC.

One downside is that QUIC is not as universally accessible as TCP
({{?RFC9283}}). This is because QUIC is built on top of UDP, which is
occasionally blocked by middleboxes.

Another downside is that QUIC is computationally expensive compared to TLS
({{?RFC8446}}) over TCP. This is partly because QUIC encrypts each packet which
is smaller than the encryption unit of TLS leading to more overhead, and partly
because UDP is less optimized in the computing infrastructure.

Due to these limitations, applications are often served on top of both QUIC and
TCP, with the former aiming to provide better user-experience, while the latter
 being considered as a backstop for network reachability or to provide
computational efficiency where necessary.

One such example is HTTP. HTTP/3 ({{?RFC9114}}) runs on top of QUIC. HTTP/2
({{?RFC9113}}) runs on top of TCP. Recently, there have been proposals to revise
HTTP/2 due to security concerns ({{?h2-stream-limits=I-D.thomson-httpbis-h2-stream-limits}}), which has led people wonder about the cost of maintaining multiple
versions of HTTP.

Another example is WebTransport. WebTransport is a super set of HTTP, but
because HTTP has different bindings for QUIC and TCP, WebTransport defines its
own bindings for the two variants of HTTP ({{?webtrans-h3=I-D.ietf-webtrans-http3}},
{{?webtrans-h2=I-D.ietf-webtrans-http2}}).

In order to eliminate the cost of these duplicated efforts to provide services
on top of both protocols, this document specifies a polyfill that allows
application protocols built on top of QUIC to run on bi-directional streams such
as TCP or TLS.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The Protocol

QUIC Services for Streams can be used on any bi-directional byte stream that is
ordered and reliable.

QUIC frames are sent directly on top of the bi-directional byte stream.

The frames are not encrypted. It is the task of the lower layer providing the
bi-directional byte stream (e.g., TLS) to provide confidentially and integrity.

QUIC packet headers are not used.

For exchanging the Transport Parameters, a new frame called TRANSPORT_PARAMETERS
frame is defined.


# QUIC Frames

In QUIC Services for Streams, the following QUIC frames are used without
modifications.

* PADDING
* PING
* RESET_STREAM
* STOP_SENDING
* STREAM (0x0a and 0x0b)
* MAX_DATA
* MAX_STREAM_DATA
* MAX_STREAMS
* DATA_BLOCKED
* STREAM_DATA_BLOCKED
* STREAMS_BLOCKED

Use of other frames defined in ({{RFC9000}}) are prohibited. If an endpoint
receives one of the prohibited frames, the endpoint MUST close the connection
with TBD error.

## STREAM Frames

In case of the STREAM frame, only the 0x0a and 0x0b variants (i.e., the ones
that carry the length but not the offset) are used.

Senders can multiplex streams, but within each stream, data MUST be sent in
order.

## TRANSPORT_PARAMETERS Frames

In QUIC Services for Streams, Transport Parameters are exchanged as frames.

This frame is the first frame being sent by an endpoint. If the first frame
being received by an endpoint is not a TRANSPORT_PARAMETERS frame, the endpoint
MUST close the connection with a TBD error.

TRANSPORT_PARAMETERS frames are formatted as show in
{{fig-transport-parameters}}.

~~~
TRANSPORT_PARAMETERS Frame {
  Type (i) = 0xTBD,
  Length (i),
  Transport Parameters (..),
}
~~~
{: #fig-transport-parameters title="TRANSPORT_PARAMETERS Frame Format}

TRANSPORT_PARAMETER frames contain the following fields:

Length:

: A variable-length integer specifying the length of the Transport Parameters
  field in this TRANSPORT_PARAMETERS frame.

Transport Parameters:

: The Transport Parameters. The encoding of the payload is as defined in
  {{Section 18 of RFC9000}}.


## Extension Frames

TBD


# Security Considerations

TODO Security


# IANA Considerations

TODO


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.