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

In order to reduce or eliminate the cost of these duplicated efforts to provide
services on top of both protocols, this document specifies a polyfill that
allows application protocols built on top of QUIC to run on bi-directional
streams such as TCP or TLS.

The polyfill being specified provides a compatibility layer for providing set of
the operations (i.e., API) required by QUIC, as specified in {{Section 2.4 and
Section 5.3 of RFC9000}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The Protocol

QUIC Services for Streams can be used on any bi-directional byte stream that is
ordered and reliable.

QUIC frames are sent directly on top of the bi-directional byte stream.

The frames are not encrypted. It is the task of the lower layer providing the
bi-directional byte stream (e.g., TLS) to provide confidentially and integrity.

QUIC packet headers are not used.

For exchanging the Transport Parameters, a new frame called
QSS_TRANSPORT_PARAMETERS frame is defined.


# QUIC Frames

In QUIC Services for Streams, the following QUIC frames are used without
modifications, as if they were send or received in the application packet number
space.

* PADDING
* RESET_STREAM
* STOP_SENDING
* STREAM (0x0a and 0x0b)
* MAX_DATA
* MAX_STREAM_DATA
* MAX_STREAMS
* DATA_BLOCKED
* STREAM_DATA_BLOCKED
* STREAMS_BLOCKED
* CONNECTION_CLOSE

Use of other frames defined in {{RFC9000}} is prohibited. If an endpoint
receives one of the prohibited frames, the endpoint MUST close the connection
with an error of type FRAME_ENCODING_ERROR.


## STREAM Frames

In this specification, only the 0x0a and 0x0b variants of the STREAM frame are
allowed (i.e., the frame format that omits the Offset field but retains the
Length field).

Senders MUST send stream payload in order, omitting the Offset field of the
STREAM frames.

Receivers retain the total amount of bytes being received for each stream, and
when receiving a STREAM frame, uses that value to determine the offset of the
newly received STREAM frame.

Use of the Length field is mandated, because QUIC Services for Streams operates
on top of bi-directional streams and the packet boundary is not observable.


## QSS_TRANSPORT_PARAMETERS Frames

In QUIC Services for Streams, Transport Parameters are exchanged as frames.

This frame is the first frame being sent by an endpoint. If the first frame
being received by an endpoint is not a QSS_TRANSPORT_PARAMETERS frame, the
endpoint MUST close the connection with an error of type
TRANSPORT_PARAMETER_ERROR.

QSS_TRANSPORT_PARAMETERS frames are formatted as show in
{{fig-transport-parameters}}.

~~~
QSS_TRANSPORT_PARAMETERS Frame {
  Type (i) = 0x3f5153300d0a0d0a,
  Length (i),
  Transport Parameters (..),
}
~~~
{: #fig-transport-parameters title="QSS_TRANSPORT_PARAMETERS Frame Format}

QSS_TRANSPORT_PARAMETERS frames contain the following fields:

Length:

: A variable-length integer specifying the length of the Transport Parameters
  field in this QSS_TRANSPORT_PARAMETERS frame.

Transport Parameters:

: The Transport Parameters. The encoding of the payload is as defined in
  {{Section 18 of RFC9000}}.


The frame type (0x3f5153300d0a0d0a; "\xffQS0\r\n\r\n" on wire) has been chosen
so that it can be used to disambiguate QUIC Services for Streams from HTTP/1.1
({{?RFC9112}}) and HTTP/2.


## QSS_PING Frames

In QUIC Services for Streams, QSS_PING frames allows endpoints to test peer
reachability above the underlying byte stream.

QSS_PING frames are formatted as shown in {{fig-qss-ping}}.

~~~
QSS_PING Frame {
  Type (i) = 0xTBD..0xTBD+1,
  Sequence Number (i),
}
~~~
{: #fig-qss-ping title="QSS_PING Frame Format"}

Type 0xTBD is used for sending a ping (i.e., request the peer to respond). Type
0xTBD+1 is used in response.

QSS_PING frames contain the following fields:

Sequence Number:

: A variable-length integer used to identify the ping.

When sending TBD_PING frames of type 0xTBD, endpoints MUST send monotonically
increasing values in the Sequence Number field, since that allows the endpoints
to identify to which ping the peer has responded.

When sending TBD_PING frames of type 0xTBD+1 in response, endpoints MUST echo
the Sequence Number that they have received.

When receiving multiple TBD_PING frames of type 0xTBD before having the chance
to respond, an endpoint MAY only respond with one TBD_PING frame of type 0xTBD+1
carrying the largest Sequence Number that the endpoint has received.


## Extension Frames

TBD


# Transport Parameters

QUIC Services for Streams uses a subset of Transport Parameters defined in
{{RFC9000}}. Also, one new Transport Parameter specific to QUIC Services for
Streams is defined.

## Permitted and Forbidden Transport Parameters

In QUIC Services for Streams, use of the following Transport Parameters is
allowed.

* max_idle_timeout
* initial_max_data
* initial_max_stream_data_bidi_local
* initial_max_stream_data_bidi_remote
* initial_max_stream_data_uni
* initial_max_streams_bidi
* initial_max_streams_uni

The definition of these frames are unchanged.

Use of other Transport Parameters defined in {{RFC9000}} is prohibited. When an
endpoint receives one of the prohibited Transport Parameters, the endpoint MUST
close the connection with an error of type TRANSPORT_PARAMETER_ERROR.

Endpoint MUST NOT send Transport Parameters that extend QUIC version 1, unless
they are specified to be compatible with QUIC Services for Streams.

When receiving Transport Parameters not defined in QUIC version 1, receivers
MUST ignore them unless they are specified to be usable on QUIC Services for
Streams.


## max_frame_size Transport Parameter

The `max_frame_size` Transport Parameter (0xTBD) is a variable-length integer
value specifying the maximum size of the QUIC frame that the peer can send, in
the unit of bytes.

The initial value of the `max_frame_size` Transport Parameter is 16384.

The maximum frame size can only be increased by sending the Transport Parameter.
It cannot be decreased. When receiving a value below the minimum, receivers MUST
close the connection with an error of type TRANSPORT_PARAMETER_ERROR.

Endpoints MUST NOT send QUIC frames that exceed the maximum declared by the
peer.

When receiving QUIC frames that exceed the declared maximum, receivers MUST
close the connection with an error of type FRAME_ENCODING_ERROR.


# Closing the Connection

As is with QUIC version 1 ({{RFC9000}}), a connection can be closed either by a
CONNECTION_CLOSE frame or by an idle timeout.

Unlike QUIC version 1, there is no draining period; once an endpoint sends or
receives the CONNECTION_CLOSE frame or reaches the idle timeout, all the
resources allocated for the Service are freed and the underlying stream is
closed immediately.


# Using 0-RTT

TLS 1.3 ({{?RFC8446}}) introduced the concept of early data (also knows as
0-RTT data).

When using QUIC Services for Streams on top of TLS that supports early data,
clients MAY use early data when resuming a connection, by reusing certain
Transport Parameters as defined in {{Section 7.4.1 of RFC9000}}.

Similarly, when accepting early data, the servers MUST send Transport Parameters
that obey to the restrictions defined in {{Section 7.4.1 of RFC9000}}.


# Security Considerations

TODO Security


# IANA Considerations

TODO


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
