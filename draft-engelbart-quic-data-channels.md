---
title: "QUIC Data Channels"
docname: draft-engelbart-quic-data-channels-latest
category: std
date: {DATE}

ipr: trust200902
area: "Applications and Real-Time"
workgroup: ""
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]
submissiontype: IETF

author:
 -
    ins: M. Engelbart
    name: Mathis Engelbart
    organization: Technical University Munich
    email: mathis.engelbart@gmail.com
 -
    ins: J. Ott
    name: Jörg Ott
    organization: Technical University Munich
    email: ott@in.tum.de


--- abstract

WebRTC data channels provide data communication between peers with varying
reliability and ordering properties. WebRTC data channels traditionally use SCTP
layered on top of DTLS over UDP. This document describes how WebRTC data
channels can be layered over QUIC.

--- middle

# Introduction

The WebRTC framework is a collection of protocols and APIs to enable endpoints
to communicate using audio and video streams and arbitrary data. In WebRTC,
real-time media and data communication happen in parallel over two independent
connections and transport protocols. While media communication in WebRTC uses
SRTP over UDP, WebRTC data channels allow two endpoints to exchange data over a
secure, multiplexed, bidirectional channel using SCTP layered on top of DTLS
over UDP. Data channels are streams of framed messages, which can be ordered or
unordered and support different reliability models. While the connection
establishment shares some aspects between media and data communication, others,
such as congestion control, transmission prioritization, and encryption, happen
independently in the stacks of the involved protocols (SCTP, DTLS, SRTP).

QUIC is a new, secure, multiplexed transport providing reliable communication
over uni- and bidirectional streams and unreliable datagrams. With the
development of RTP over QUIC (RoQ), there is already an alternative for media
communication using QUIC instead of SRTP for RTP media communication. One of the
main requirements during the development of RoQ was the ability to share a
single QUIC connection for media communication and arbitrary data exchange.
Using QUIC as a common protocol for media and data would allow for easy sharing
of encryption context, congestion control, and prioritization among all
transmissions between two endpoints. While RoQ includes capabilities for
multiplexing RTP, RTCP, and other protocols in one QUIC connection, no such
protocol is currently defined.

This document aims to fill this gap by specifying a data communication protocol
similar to WebRTC data channels that can run over a QUIC connection, multiplexed
with RoQ. The protocol defined in this document is referred to as QUIC Data
Channels (QDC).

WebRTC data channels have several use cases and requirements defined in
{{!RFC8831}}. The data channel protocol defined in this protocol tries to
fulfill the same requirements.

The remainder of this document is organized as follows:

# Connection Establishment

QUIC requires the use of ALPN {{!RFC7301}} during connection setup. Data
channels over QUIC use the "qdc" token during the TLS handshake.

## Draft version identification

> **RFC Editor's note:** Please remove this section prior to publication of a
> final version of this document.

Data channels over QUIC use "qdc" as the ALPN identifier.

Only implementations of the final, published RFC can identify themselves as
"qdc". Until such an RFC exists, implementations MUST NOT identify themselves
using this string.

Implementations of draft versions of the protocol MUST add the string "-" and
the corresponding draft number to the identifier. For example,
draft-engelbart-quic-data-channels-04 is identified using the string "qdc-04".

Non-compatible experiments based on these draft versions MUST append the string
"-" and an experiment name to the identifier.

# Messages and QUIC Streams

QDC uses a message abstraction on top of QUIC streams. To provide framed
messages to the application, QDC uses a single QUIC stream for each message. An
endpoint MUST close a QUIC stream after sending a message on the stream. Message
framing is implicitly provided by QUIC streams. Messages sizes are only limited
by the maximum size of a QUIC stream.

# The Data Channel Establishment Protocol

The data channel establishement protocol (DCEP) is defined in {{!RFC8832}}. DCEP
provides functionality to setup data channels between two peers without external
signaling. QDC uses a similar protocol to establish new data channels. To open a
new data channel, an endpoint sends a Data Channel Open Message.

The Open Message includes a channel identifier that will be used to uniquely
identify the data channel. To avoid collisions where both sides try to open a
data channel with the same stream identifiers, each side MUST use streams with
either even or odd stream identifiers when sending a Data Channel Open Message.
When using QUIC, the method used to determine which side uses odd or even is
based on the underlying DTLS connection role: the side acting as the QUIC client
MUST use streams with even stream identifiers; the side acting as the QUIC
server MUST use streams with odd stream identifiers.

# Message Ordering

Data channels are streams of ordered or unordered messages. The type of the data
channel determines if the messages are ordered or unordered. Data Messages on
ordered data channels have an additional Sequence Number, that a receiver can
use to reorder messages that arrived out of order. Each message of an ordered
data channel MUST carry the sequence number. Messages on unordered data channels
MUST NOT carry sequence numbers. Endpoints MUST treat messages without sequence
number on an ordered channel as an error of type PROTOCOL_VIOLATION. Endpoints
SHOULD treat messages with a sequence number on an unordered data channel as an
error of type PROTOCOL_VIOLATION.

# Message Priority {#message-priority}

> **TODO**

# Partial Reliability

Data channels support two partial reliability modes. Reliability can either be
limited in the number of retransmissions or it can be expressed as a deadline
after which no further retransmissions will occur. This section explains how
these modes are supported by leveraging the capabilities of the underlying QUIC
connection.

## Limited Number of Retransmissions

> **TODO:** How can this be implemented over QUIC? Do we need it?
> Datagrams are an option, but that would limit the message size to MTU size or
> require fragmentation over datagrams.
> It could also be an API requirement to the underlying QUIC stack, but that
> may not be easy to get support for.

## Limited Duration of Retransmissions

The sender can limit the lifetime of a message on a data channel. The lifetime
is defined per data channel and is the same for every message sent on the
channel. The lifetime of the messages on a data channel is communicated in the
*Reliability Parameter* of the Data Channel Open Messages (see
{{data-channel-open-message}}).

The lifetime of a message starts when the sender opens a QUIC stream to send the
message on. The lifetime ends after the amount of time that was announced in the
*Reliability Parameter*. When the lifetime of a message expires, the sender of
the message SHOULD close the stream using QUIC's RESET_STREAM frame.

> **TODO:** Instead of RESET_STREAM, one could use CLOSE_STREAM with an offset
> after the message header, so that the receiver knows which channel the message
> belonged to, what type it head and optionally can read sequence number and
> length.

# Message Types and Formats

This section describes the message formats used for QUIC data channels.

## Data Channel Open Message {#data-channel-open-message}

The format of Data Channel Open Messages is depicted in
{{fig-data-channel-open-msg}}.

> **TODO:** Explain how to choose channel IDs (similar to {{Section 4 of
> RFC8832}}.

~~~
Data Channel Open Message {
  Channel ID (i),
  Message Type (i) = 0x00,
  Channel Type (8),
  Priority (i),
  Reliability Parameter (i),
  Label Length (i),
  Label (..),
  Protocol Length (i),
  Protocol (..),
}
~~~
{: #fig-data-channel-open-msg title="Data Channel Open Message"}

Channel ID:
: A unique identifier of the data channel.

Message Type:
: The Data Channel Open Message type is always set to `0x00`.

Channel Type:
: This field specifies the type of data channel to be opened. The values are
managed by IANA and follow the registry defined in {{Section 8.2.2 of
!RFC8832}}.

Priority:
: The message priority as described in {{message-priority}}.

Reliability Parameter:
: For reliable data channels, this field MUST be set to 0 on the sending side
and MUST be ignored on the receiving side. If a partially reliable data channel
with a limited number of retransmissions is used, this field specifies the
number of retransmissions. If a partially reliable data channel with a limited
lifetime is used, this field specifies the maximum lifetime in milliseconds (see
also {{Section 5.1 of !RFC8832}}).

Label Length:
: A variable-length integer specifying the length of the label field in bytes.

Label:
: The name of the data channel as a UTF-8-encoded string, as specified in
{{!RFC3629}}. This may be an empty string.

Protocol Length:
: A variable-length integer specifying the length of the protocol field in
bytes.

Protocol:
: If this is an empty string, the protocol is unspecified. If it is a non-empty
string, it specifies a protocol registered in the "WebSocket Subprotocol Name
Registry" created in {{!RFC6455}}. This string is UTF-8 encoded, as specified in
{{!RFC3629}}.

## Data Channel Close Message

~~~
Data Channel Open Message {
  Channel ID (i),
  Message Type(i) = 0x01,
}
~~~
{: #fig-data-channel-close-msg title="Data Channel Close Message"}

Channel ID:
: The unique identifier of the data channel.

Message Type:
: The Data Channel Close Message type is always set to `0x01`.

## Data Message

The Data Message has two optional fields. The message type field takes the form
`0b000001XX`. The two low-order bits determine the fields that are present in
the message:

* The SEQ bit (0x02) indicates that a sequence number is present. If this bit is
  set to 0, the Sequence Number field is absent. If this bit is set to 1, the
  Sequence Number field is present.
* The LEN bit (0x01) indicates that a Length field is present. If this bit is
  set to 0, the Length field is absent and the Message Data field extends to the
  end of the packet. If this bit is set to 1, the Length field is present.

~~~
Data Message {
  Channel ID (i),
  Message Type (i) = 0x02..0x05
  [Sequence Number (i)],
  [Length (i)],
  Message Data (..),
}
~~~
{: #fig-data-message title="A message on a reliable and unordered datachannel}

Channel ID:
: The unique identifier of the data channel.

Message Type:
: The Data Channel Open Message type is always set to `0x02`.

Sequence Number:
: A variable-length integer specifying the sequence number in the channel for
the message. This field is present when the SEQ bit is set to 1.

Length:
: A variable-length integer specifying the length of the Message Data field.
This field is present when the LEN bit is set to 1. When the LEN bit is set to
0, the Message Data field consumes all the remaining bytes in the QUIC stream.

Message Data:
: The bytes of the message to be delivered.

# Error Handling

# Security Considerations {#sec-considerations}

The security considerations for the QUIC protocol and datagram extension
described in {{Section 21 of !RFC9000}}, {{Section 9 of !RFC9001}}, {{Section 8
of !RFC9002}} and {{Section 6 of !RFC9221}} apply.

# IANA Considerations {#iana-considerations}

## ALPN Registration

## Registration of a QUIC Data Channels Identification String

This document creates a new registration for the identification of QUIC Data
Channels in the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs"
registry {{?RFC7301}}.

The "qdc" string identifies QUIC Data Channels:

  Protocol:
  : QUIC Data Channels

  Identification Sequence:
  : TODO

  Specification:
  : This document

## Error Codes Registry

--- back

# Acknowledgments
{:numbered="false"}

