---
title: "QUIC on Streams"
docname: draft-kazuho-quic-quic-on-streams-latest
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
 -  fullname: Lucas Pardue
    organization: Cloudflare
    email: lucas@lucaspardue.com

--- abstract

This document specifies a polyfill of QUIC version 1 that runs on top of
bi-directional streams such as TLS.


--- middle

# Introduction

QUIC version 1 {{!QUIC=RFC9000}} is a bi-directional, authenticated
transport-layer protocol built on top of UDP {{?UDP=RFC768}}. The protocol
provides multiplexed flow-controlled streams without head-of-line blocking as a
core service. It also offers low-latency connection establishment and efficient
loss recovery.

However, there are downsides to QUIC.

One downside is that QUIC, being based on UDP, is not as universally accessible
as TCP {{?TCP=RFC9293}}, due to occasionally being blocked by middleboxes.

Another downside is that QUIC is computationally more expensive compared to TLS
{{!TLS13=RFC8446}} over TCP. This increased cost is partly because QUIC encrypts
each packet, which is smaller than the encryption unit of TLS, leading to more
overhead, and partly because UDP is less optimized within computing
infrastructures.

Due to these limitations, applications are often served using both QUIC and TCP.
QUIC is employed to provide the optimal user experience, while TCP acts as a
fallback for ensuring network reachability and computational efficiency as
needed.

One such example is HTTP, which has different bindings for QUIC (HTTP/3
{{?HTTP3=RFC9114}}) and TCP (HTTP/2 {{?HTTP2=RFC9113}}). Recently, security
concerns have prompted proposals to revise HTTP/2
({{?h2-stream-limits=I-D.thomson-httpbis-h2-stream-limits}}), which has sparked
discussions about the costs of maintaining multiple HTTP versions.

Another example is WebTransport, a superset of HTTP. Because HTTP has different
bindings for QUIC and TCP, WebTransport defines its own extensions for the two
HTTP variants ({{?webtrans-h3=I-D.ietf-webtrans-http3}},
{{?webtrans-h2=I-D.ietf-webtrans-http2}}).

To reduce or eliminate the costs associated with duplicated efforts in providing
services on top of both transport protocols, this document specifies a polyfill
that allows application protocols built on QUIC to run on transport protocols
that provide single bi-directional, byte-oriented stream such as TCP or TLS.

The specified polyfill provides a compatibility layer for the set of operations
(i.e., API) required by QUIC, as specified in {{Section 2.4 and Section 5.3 of
QUIC}}.


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

UNIX sockets are an example that provides only the first two. Congestion control
is not employed, as UNIX sockets do not face a shared bottleneck.
Confidentiality and integrity protection are deemed unnecessary in environments
where the operating system is trusted.


# QUIC Frames

In QUIC on Streams, the following QUIC frames can be used, as if they were sent
or received in the application packet number space:

* PADDING
* RESET_STREAM
* STOP_SENDING
* STREAM
* MAX_DATA
* MAX_STREAM_DATA
* MAX_STREAMS
* DATA_BLOCKED
* STREAM_DATA_BLOCKED
* STREAMS_BLOCKED
* CONNECTION_CLOSE

The frame formats are identical to those in QUIC version 1. Likewise, the
meaning and requirements for the use of these frames are consistent with QUIC
version 1, with the exception to the specific changes made to the STREAM frames,
as detailed in {{stream-frames}}.

Use of other frames defined in QUIC version 1 is prohibited for various reasons.
ACK frames are not used because the underlying transport guarantees delivery.
Frames related to the cryptographic handshake are not used because an underlying
security layer can provide equivalent features. Use of frames that communicate
Connection IDs and those related to path migration is forbidden.

The full list of prohibited frames is:

* PING
* ACK
* CRYPTO
* NEW_TOKEN
* NEW_CONNECTION_ID
* RETIRE_CONNECTION_ID
* PATH_CHALLENGE
* PATH_REPONSE
* HANDSHAKE_DONE

Endpoints MUST NOT send prohibited frames. If an endpoint receives one it MUST
close the connection with an error of type FRAME_ENCODING_ERROR.


## STREAM Frames {#stream-frames}

While the frame format remains unchanged, there are two differences in the
handling of STREAM frames between QUIC version 1 and QUIC on Streams.


### STREAM Frames without the Length Field

In QUIC on Streams, when a STREAM frame that omits the Length field is used, the
size of that STREAM frame is determined by the maximum frame size, as regulated
by the `max_frame_size` Transport Parameter ({{max_frame_size}}).

This behavior contrasts with that of QUIC version 1, where the absence of the
Length field implies that the STREAM frame extends to the end of the QUIC packet
payload.

This variation arises due to the characteristics of the underlying transports of
QUIC on Streams, which may not have, or provide visibility into, the packet
boundaries.


### Ordering of STREAM frames

For each stream being sent, senders MUST send stream payload in order.

When receiving a STREAM frame that carries a payload not immediately following
the payload of the previous STREAM frame for the same Stream ID, receivers MUST
close connection with an error of type PROTOCOL_VIOLATION_ERROR.

This change from QUIC version 1 eliminates the need for implementations to
buffer and reassemble the stream payload. As a result, the payload being
received can be directly passed to the application as it is read from the
transport. This efficiency is due to the underlying transport's guarantee of
in-order delivery.

These changes do not impact the senders' capability to interleave STREAM frames
from multiple streams.


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
  {{Section 18 of QUIC}}.


The QS_TRANSPORT_PARAMETERS frame is the first frame being sent by endpoints.
Endpoints MUST send the QS_TRANSPORT_PARAMETERS frame as soon as the underlying
transport becomes available. Note neither endpoint needs to wait for the
peer's Transport Parameters before sending its own, as Transport Parameters are
a unilateral declaration of an endpoint's capabilities
({{Section 7.4 of QUIC}}).

If the first frame being received by an endpoint is not a
QS_TRANSPORT_PARAMETERS frame, the endpoint MUST close the connection with an
error of type TRANSPORT_PARAMETER_ERROR.

The frame type (0x3f5153300d0a0d0a; "\xffQS0\r\n\r\n" on wire) has been chosen
so that it can be used to disambiguate QUIC on Streams from HTTP/1.1
{{?HTTP1=RFC9112}} and HTTP/2.


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

QUIC on Streams uses a subset of Transport Parameters defined in QUIC version 1.
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

Use of other Transport Parameters defined in QUIC version 1 is prohibited. When
an endpoint receives one of the prohibited Transport Parameters, the endpoint
MUST close the connection with an error of type TRANSPORT_PARAMETER_ERROR.

Endpoints MUST NOT send Transport Parameters that extend QUIC version 1, unless
they are specified to be compatible with QUIC on Streams.

When receiving Transport Parameters not defined in QUIC version 1, receivers
MUST ignore them unless they are specified to be usable on QUIC on Streams.


## max_frame_size Transport Parameter {#max_frame_size}

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

As is with QUIC version 1, a connection can be closed either by a
CONNECTION_CLOSE frame or by an idle timeout.

Unlike QUIC version 1, there is no draining period; once an endpoint sends or
receives the CONNECTION_CLOSE frame or reaches the idle timeout, all the
resources allocated for the Service are freed and the underlying transport is
closed immediately.


# Using 0-RTT

TLS 1.3 introduced the concept of early data (also knows as 0-RTT data).

When using QUIC on Streams on top of TLS that supports early data, clients MAY
use early data when resuming a connection, by reusing certain Transport
Parameters as defined in {{Section 7.4.1 of QUIC}}.

Similarly, when accepting early data, the servers MUST send Transport Parameters
that obey to the restrictions defined in {{Section 7.4.1 of QUIC}}.


# Extensions

Not all the extensions of QUIC version 1 can be used. Each extension have to
define its mapping for QUIC on Streams, or explicitly allow the use; see
{{permitted-tps}}.

As is the case with QUIC version 1, use of extension frames have to be
negotiated before use; see {{Section 19.21 of QUIC}}.

This specification defines the mapping of the Unreliable Datagram Extension.


## Unreliable Datagram Extension

The use of the Unreliable Datagram Extension {{!QUIC_DATAGRAM=RFC9221}} is
permitted, with one modification:

Similar to STREAM frames, when employing DATAGRAM frames of type 0x30 (i.e.,
DATAGRAM frames without the Length field), their size is determined by the
`max_frame_size` Transport Parameter ({{max_frame_size}}).

Apart from this, the encoding and semantics of the Unreliable Datagram Extension
remain unchanged. The use of the extension is negotiated via the Transport
Parameters.

As discussed in {{Section 5 of QUIC_DATAGRAM}}, senders can drop DATAGRAM frames
if the transport is blocked by flow or congestion control.


# Version Agility

Unlike QUIC, QUIC on Streams does not define a mechanism for version
negotiation.

In large-scale deployments requiring service and protocol version discovery,
QUIC on Streams can and is likely to be implemented over TLS. The
Application-Layer Protocol Negotiation Extension of TLS {{?ALPN=RFC7301}} is the
favored mechanism to negotiate between an application protocol based on this
specification and others.

When ALPN is unavailable, first 8 bytes exchanged on the transport (i.e., the
type field of the QS_TRANSPORT_PARAMETERS frame in the encoded form) can be used
to identify if QUIC on Streams is in use.


# Implementation Considerations

Similar to HTTP/3 with Extensible Priorities {{?HTTP_PRIORITY=RFC9218}},
application protocols using QUIC may employ stream multiplexing along with a
system to tune the delivery sequence of QUIC streams.

To alternate between QUIC streams of varying priorities in a timely manner, it
is advisable for QUIC on Streams implementations to avoid creating deep buffers
holding QUIC frames. Instead, endpoints should wait for the transport layer to
be ready for writing. Upon becoming writable, they should write QUIC frames
according to the latest prioritization signals.

Additionally, implementations may consider monitoring or adjusting the flow and
congestion control parameters of the underlying transport. This approach aims to
minimize data buffering within the transport layer before transmission. However,
improper adjustment of these parameters could potentially lead to lower
throughput.


# Security Considerations

TODO Security


# IANA Considerations

TODO


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
