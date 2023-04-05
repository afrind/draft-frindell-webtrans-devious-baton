---
title: "Devious Baton Protocol for Exercising WebTransport"
category: info

docname: draft-frindell-webtrans-devious-baton-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "WebTransport"
keyword:
 - WebTransport
venue:
  group: "WebTransport"
  type: "Working Group"
  mail: "webtransport@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/webtransport/"
  github: "afrind/draft-frindell-webtrans-devious-baton"
  latest: "https://afrind.github.io/draft-frindell-webtrans-devious-baton/draft-frindell-webtrans-devious-baton.html"

author:
 -
    fullname: Alan Frindell
    organization: Meta
    email: "afrind@meta.com"

normative:

informative:


--- abstract

This document describes a simple protocol that can be used to exercise the
functionality provided by WebTransport.  The protocol passes a "baton" between
endpoints, using both unidirectional and bidirectional streams.

--- middle

# Introduction

WebTransport offers applications the ability to send and receive data over
bidirectional and unidirectional streams, as well as send and received
datagrams.  This protocol can be used to test the full suite of functionality in
a WebTransport implementations and demonstrate interoperability.

The protocol works by passing a "baton" -- a one byte integer -- between
endpoints using streams.  A receiving endpoint increments the baton value modulo
256 and sends it to the peer until the baton's value reaches 0.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Session Establishment

The client initiates a WebTransport session as defined in [webtransport].  The
protocol can be used by any endpoint, but for interoperability it is RECOMMENDED
that the URL path be /webtransport/devious-baton.

## Query Parameters

The server MUST support the following optional query parameters:

* version - an integer specifying the draft version of Devious Baton the client
  intends to use

If the version is invalid or the server does not support the specified version,
it MUST reject the WebTransport session with a 4xx status code.  The default
value is 0.

* baton - an integer between 0 and 255, inclusive, which the server will use as
  the initial baton value

If the baton value is invalid, the server MUST reject the WebTransport session
with a 4xx status code.  There is no default - if unspecified the server chooses
a random baton value.

* count - an integer specifying how many batons will be sent in parallel

The default value it 1.  If the client asks for more batons than the server is
capabale of sending, the server MUST reject the WebTransport session with a 4xx
status code

# Protocol Behavior

## Setup

Upon successful negotiation of a WebTransport session to the Devious Baton
endpoint, the server opens a unidirectional stream for each baton.  If there is
insufficient stream credit to open a unidirectional stream, the server MUST
close the WebTransport session with the INSUFFICIENT_STREAMS error code.  The
server sends a Baton message with the initial baton value on each stream and
closes it.

## Processing a Baton Message

When either endpoint receives a Baton message on a stream, it takes the
following actions:

* If the value of the baton is 0, it closes the WebTransport session with no
  error
* Sets a new baton value to the old baton value + 1 modulo 256
* Sends a Baton message on a stream
* Closes the stream

The stream to send the outgoing Baton message on depends on how the incoming
Baton message arrived.

* If the incoming Baton message arrived on a unidirectional stream, the endpoint
  opens a bidirectional stream and sends the outgoing Baton message on it.  * If
  the Baton message arrived on a peer-initiated bidirectional stream, the
  endpoint sends the outgoing Baton message on that stream
* If the Baton message arrived on a self-initiated bidirectional stream, the
  endpoint opens a unidirectional stream and sends the outgoing Baton message on
  it.

If the endpoint has insufficient stream credit to open the correct type of
stream, it MUST close the WebTransport session with the INSUFFICIENT_STREAMS
error code.

## Baton message

```
Baton Message {
  padding length(i)
  padding(...)
  baton(1)
}
```

To allow for exercising of long streams and flow control, the Baton message
begins with an aribtrary amount of padding.  `padding length` specifies the
number of bytes of padding.  The `padding` field contains `padding length`
octets of padding.  The receiver ignores the bytes themselves so they can be any
value, for example 0 or random data.

`baton` contains the current value of the baton.  It is a single byte to enforce
the modulo 256 arithmetic.

## Datagrams

When a client endpoint receives a Baton message with a baton value = 1 modulo 7,
it sends a datagram with an identical Baton message.  When a server endpoint
receives a Baton message with a baton value = 0 modulo 7, it sends a datagram
with an identical Baton message. Note that a Baton message in a datagram MUST
use a padding value small enough such that the entire Baton message fits in a
single datagram.

## Error Handling

If an endpoint receives a gracefully closed stream or datagram with an
incomplete Baton message, it MUST close the WebTransport session with the BRUH
error code.

Either endpoint can send a STOP_SENDING or RESET_STREAM on an open stream.  Upon
receipt of a STOP_SENDING on a stream, the endpoint MUST send a RESET_STREAM for
that stream unless it has already closed the stream.  If an endpoint detects
that all baton streams have been reset, it MUST close the WebTransport session
with the GAME_OVER error code.

If an endpoint gets tired of waiting for the next Baton message, it MAY close
the WebTransport session with the BORED error code.

### Error Codes

| Name                 |  Code  | Description                         |
| -------------------- | :----: | ----------------------------------- |
| INSUFFICIENT_STREAMS |  0x01  | There is insufficient stream credit to continue the protocol |
| BRUH                 |  0x02  | Received a malformed Baton message  |
| GAME_OVER            |  0x03  | All baton streams have been reset   |
| BORED                |  0x04  | Got tired of waiting for the next message |

# Security Considerations

There are not believed to be any further security considerations beyond those
presented in QUIC Transport.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

Martin Thomson, Christian Huitema and Lucas Pardue contributed ideas to this
protocol.  David Schinazi suggested the name Devious Baton.
