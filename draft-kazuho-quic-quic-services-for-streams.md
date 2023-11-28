---
title: "QUIC on Streams"
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
({{?RFC9293}}). This is because QUIC is built on top of UDP, which is
occasionally blocked by middleboxes.

Another downside is that QUIC is computationally expensive compared to TLS
({{!RFC8446}}) over TCP. This is partly because QUIC encrypts each packet which
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
services on top of both transport protocols, this document specifies a polyfill
that allows application protocols built on top of QUIC to run on transport
protocols providing single bi-directional, byte-oriented stream such as TCP or
TLS.

The polyfill being specified provides a compatibility layer for providing set of
the operations (i.e., API) required by QUIC, as specified in {{Section 2.4 and
Section 5.3 of RFC9000}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The Protocol

QUIC on Streams can be used on any transport that provides bi-directional,
byte-oriented stream that is ordered and reliable; for details, see
{{transport-properties}}.

QUIC frames are sent directly on top of the transport.

The frames are not encrypted. It is the task of the transport (e.g., TLS) to
provide confidentially and integrity.

QUIC packet headers are not used.

For exchanging the Transport Parameters, a new frame called
QS_TRANSPORT_PARAMETERS frame is defined.


## Properties of Underlying Transport {#transport-properties}

QUIC on Streams is designed to work on top of transport layer protocols that
provide the following capabilities:

In-order delivery of bytes in both direction:

: Underlying transport provides a byte-oriented and bi-directional stream that
  deliver the bytes in order; i.e., bytes that were sent in one order become
  available to the receiving side in the same order.

Guaranteed delivery:

: If the transport runs on top of a lossy network, that transport recovers the
  bytes lost; e.g., by retransmitting them. This requires buffering and
  reassembly, in order to achieve the first bullet point (in-order delivery).

Congestion control:

: When used on a shared network, the transport is congestion controlled.
  Implementations of QUIC on Streams simply write outgoing frames to the
  transport when that transport permits to.

Confidentially and Integrity:

: Unless used upon endpoints between which tampering or monitoring is a
  non-concern, the transport provides confidentially and integrity protection.

TLS over TCP provides all these capabilities.

UNIX sockets is an example that provides only the first two. Congestion control
is not used because UNIX sockets do not work on top of a shared network.
Confidentiallity and integrity protection is considered unnecessary when the
operating system can be trusted.


# QUIC Frames

In QUIC on Streams, the following QUIC frames can be used, as if they were sent
or received in the application packet number space:

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

The format and the meaning of these frames are unchanged, with the STREAM frames
being an exception. For the details of the STREAM frames, see {{stream-frames}}.

Use of other frames defined in {{RFC9000}} is prohibited. Namely, ACK frames are
not used, because the underlying transport guarantees delivery. Use of frames
that communicate Connection IDs and those related to path migration is
forbidden.

If an endpoint receives one of the prohibited frames, the endpoint MUST close
the connection with an error of type FRAME_ENCODING_ERROR.


## STREAM Frames {#stream-frames}

In this specification, only the 0x0a and 0x0b variants of the STREAM frame are
allowed (i.e., the frame format that omits the Offset field but retains the
Length field).

Senders MUST send stream payload in order, omitting the Offset field of the
STREAM frames.

Receivers retain the total amount of bytes being received for each stream, and
when receiving a STREAM frame, uses that value to determine the offset of the
newly received STREAM frame.

Unlike QUIC version 1, receivers do not need to buffer and reassemble the
payload of each incoming stream. This is because the sender sends the STREAM
frames in order and that they will be delivered in-order by the transport. The
payload being received can be passed to the application as they are read from
the transport.

Use of the Length field is mandated, because QUIC on Streams operates on top of
byte-oriented transports and thus the packet boundary may not be
observable.


## QS_TRANSPORT_PARAMETERS Frames

In QUIC on Streams, Transport Parameters are exchanged as frames.

QS_TRANSPORT_PARAMETERS frames are formatted as shown in
{{fig-qs-transport-parameters}}.

~~~
QS_TRANSPORT_PARAMETERS Frame {
  Type (i) = 0x3f5153300d0a0d0a,
  Length (i),
  Transport Parameters (..),
}
~~~
{: #fig-qs-transport-parameters title="QS_TRANSPORT_PARAMETERS Frame Format"}

QS_TRANSPORT_PARAMETERS frames contain the following fields:

Length:

: A variable-length integer specifying the length of the Transport Parameters
  field in this QS_TRANSPORT_PARAMETERS frame.

Transport Parameters:

: The Transport Parameters. The encoding of the payload is as defined in
  {{Section 18 of RFC9000}}.


The QS_TRANSPORT_PARAMETERS frame is the first frame being sent by endpoints.
Endpoints MUST send the QS_TRANSPORT_PARAMETERS frame as soon as the underlying
transport becomes available. Note neither endpoint needs to wait for the
peer's Transport Parameters before sending its own, as Transport Parameters are
a unilateral declaration of an endpoint's capabilities
({{Section 7.4 of RFC9000}}).

If the first frame being received by an endpoint is not a
QS_TRANSPORT_PARAMETERS frame, the endpoint MUST close the connection with an
error of type TRANSPORT_PARAMETER_ERROR.

The frame type (0x3f5153300d0a0d0a; "\xffQS0\r\n\r\n" on wire) has been chosen
so that it can be used to disambiguate QUIC on Streams from HTTP/1.1
({{?RFC9112}}) and HTTP/2.


## QS_PING Frames

In QUIC on Streams, QS_PING frames allow endpoints to test peer reachability
above the underlying transport.

QS_PING frames are formatted as shown in {{fig-qs-ping}}.

~~~
QS_PING Frame {
  Type (i) = 0xTBD..0xTBD+1,
  Sequence Number (i),
}
~~~
{: #fig-qs-ping title="QS_PING Frame Format"}

Type 0xTBD is used for sending a ping (i.e., request the peer to respond). Type
0xTBD+1 is used in response.

QS_PING frames contain the following fields:

Sequence Number:

: A variable-length integer used to identify the ping.

When sending QS_PING frames of type 0xTBD, endpoints MUST send monotonically
increasing values in the Sequence Number field, since that allows the endpoints
to identify to which ping the peer has responded.

When sending QS_PING frames of type 0xTBD+1 in response, endpoints MUST echo the
Sequence Number that they received.

When receiving multiple QS_PING frames of type 0xTBD before having the chance to
respond, an endpoint MAY only respond with one QS_PING frame of type 0xTBD+1
carrying the largest Sequence Number that the endpoint has received.


# Transport Parameters

QUIC on Streams uses a subset of Transport Parameters defined in {{RFC9000}}.
Also, one new Transport Parameter specific to QUIC on Streams is defined.

## Permitted and Forbidden Transport Parameters {#permitted-tps}

In QUIC on Streams, use of the following Transport Parameters is allowed.

* max_idle_timeout
* initial_max_data
* initial_max_stream_data_bidi_local
* initial_max_stream_data_bidi_remote
* initial_max_stream_data_uni
* initial_max_streams_bidi
* initial_max_streams_uni

The definition of these Transport Parameters are unchanged.

Use of other Transport Parameters defined in {{RFC9000}} is prohibited. When an
endpoint receives one of the prohibited Transport Parameters, the endpoint MUST
close the connection with an error of type TRANSPORT_PARAMETER_ERROR.

Endpoints MUST NOT send Transport Parameters that extend QUIC version 1, unless
they are specified to be compatible with QUIC on Streams.

When receiving Transport Parameters not defined in QUIC version 1, receivers
MUST ignore them unless they are specified to be usable on QUIC on Streams.


## max_frame_size Transport Parameter

The `max_frame_size` Transport Parameter (0xTBD) is a variable-length integer
specifying the maximum size of the QUIC frame that the peer can send, in the
unit of bytes.

The initial value of the `max_frame_size` Transport Parameter is 16384.

By sending the Transport Parameter, the maximum frame size can only be
increased. When receiving a value below the initial value, receivers MUST close
the connection with an error of type TRANSPORT_PARAMETER_ERROR.

Endpoints MUST NOT send QUIC frames that exceed the maximum declared by the
peer.

When receiving QUIC frames that exceed the declared maximum, receivers MUST
close the connection with an error of type FRAME_ENCODING_ERROR.


# Closing the Connection

As is with QUIC version 1 ({{RFC9000}}), a connection can be closed either by a
CONNECTION_CLOSE frame or by an idle timeout.

Unlike QUIC version 1, there is no draining period; once an endpoint sends or
receives the CONNECTION_CLOSE frame or reaches the idle timeout, all the
resources allocated for the Service are freed and the underlying transport is
closed immediately.


# Using 0-RTT

TLS 1.3 ({{?RFC8446}}) introduced the concept of early data (also knows as
0-RTT data).

When using QUIC on Streams on top of TLS that supports early data, clients MAY
use early data when resuming a connection, by reusing certain Transport
Parameters as defined in {{Section 7.4.1 of RFC9000}}.

Similarly, when accepting early data, the servers MUST send Transport Parameters
that obey to the restrictions defined in {{Section 7.4.1 of RFC9000}}.


# Extensions

Not all the extensions of QUIC version 1 can be used. Each extension have to
define its mapping for QUIC on Streams, or explicitly allow the use; see
{{permitted-tps}}.

As is the case with QUIC version 1, use of extension frames have to be
negotiated before use; see {{Section 19.21 of RFC9000}}.

This specification defines the mapping of the Unreliable Datagram Extension.


## Unreliable Datagram Extension

Use of the Unreliable Datagram Extension ({{!RFC9221}}) is possible, with the
following changes.

When using the DATAGRAM frame, senders MUST use only the variant of type 0x31
(i.e., DATAGRAM frame with the Length field). If an endpoint receives a DATAGRAM
frame of type 0x30 (i.e., DATAGRAM frame without he Length field), the endpoint
MUST close the connection with an error of type FRAME_ENCODING_ERROR.

Otherwise, the encoding and the semantics of the DATAGRAM frame as well as the
`max_datagram_frame_size` Transport Parameter is unchanged. Use of the
Unreliable Datagram Extension is negotiated using the Transport Parameter.

As discussed in {{Section 5 of RFC9221}}, DATAGRAM frames can be dropped if the
sending side of the transport is blocked by flow or congestion control.


# Version Agility

Unlike QUIC, QUIC on Streams does not define a mechanism for version
negotiation.

In large-scale deployments that require service and protocol version discovery,
QUIC on Streams can and is likely to be used on top of TLS. ALPN ({{?RFC7301}})
is the preferred mechanism to negotiate between an application protocol built on
top of this specification and others.

When ALPN is unavailable, first 8 bytes exchanged on the transport (i.e., the
type field of the QS_TRANSPORT_PARAMETERS frame in the encoded form) can be used
to identify if QUIC on Streams is in use.


# Implementation Considerations

Like HTTP/3 ({{?RFC9114}}) with Extensible Priorities ({{?RFC9218}}),
application protocols built on top of QUIC might use stream multiplexing in
conjunction with a mechanism to request or specify the order in which the
payload of the QUIC streams are to be delivered.

To switch between QUIC streams with different priorities in a timely manner,
implementations of QUIC on Streams should refrain from building deep buffers
that contain QUIC frames to be sent in particular order. Rather,
endpoints are encouraged to wait for the underlying transport to become
writable, and each time it becomes writable, write new frames based on the most
recent prioritization signals.

Implementations might also observe or tune the values of underlying transports
related to flow and congestion control, in order to minimize the amount of data
buffered inside the transport layer without immediately being sent. Note however
that failures to tune these variables might lead to reduced throughput.


# Security Considerations

TODO Security


# IANA Considerations

TODO


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
