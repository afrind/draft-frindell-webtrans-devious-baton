---
title: "Devious Baton Protocol for Exercising WebTransport"
category: info

docname: draft-frindell-webtrans-devious-baton-latest
abbrev: devious-baton
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
datagrams.  The Devious Baton protocol is an application that can be used to
test the full suite of functionality in a WebTransport implementation and
demonstrate interoperability.

The protocol works by passing a "baton" -- a one byte integer -- between
endpoints using streams.  A receiving endpoint increments the baton value modulo
256 and sends it to the peer until the baton's value reaches 0.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Client:

: The endpoint that initiates the WebTransport session

Server:

: The endpoint that did not initiate the WebTransport session

Devious Baton Session:

: A single WebTransport session initiated as described in
 {{session-establishment}}

# Session Establishment

The client initiates a WebTransport session as defined in
{{!OVERVIEW=I-D.ietf-webtrans-overview}}.  The protocol can be used by
any endpoint, but for interoperability it is RECOMMENDED that the URL
path be /webtransport/devious-baton.

## Query Parameters

The behavior of the protocol can be configured by parameters indicated by the
client. These parameters are transmitted in query parameters in the session
establishment URL. Sending parameters is optional but omission of a parameter
requires the server to interpret that as the default value.

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

The default value is 1.  If the client asks for more batons than the server is
capable of sending, the server MUST reject the WebTransport session with a 4xx
status code

## Protocol Version

This draft defines Devious Baton protocol version 0.

# Protocol Behavior

## Setup

Upon successful negotiation of a WebTransport session to the Devious Baton
endpoint, the server opens a unidirectional stream for each baton.  If there is
insufficient stream credit to open a unidirectional stream, the server MUST
close the WebTransport session with the DA_YAMN session error code.
The server sends a Baton message with the initial baton value on each stream and
closes it.

## Processing a Baton Message

When either endpoint receives a Baton message on a stream, it takes the
following actions:

* If the value of the baton is 0, the endpoint MUST close the WebTransport
  session with no error
* If the value of the baton is not 0, the endpoint MUST send a new Baton message
  with a baton value equal to the incoming baton value + 1 modulo 256.  The new
  Baton message is sent on a stream, decribed below.
* After sending the Baton message, the endpoint MUST close the stream

The endpoint selects the outgoing Baton message stream based on how the incoming
Baton message arrived.

* If the incoming Baton message arrived on a unidirectional stream, the endpoint
  opens a bidirectional stream and sends the outgoing Baton message on it.
* If the Baton message arrived on a peer-initiated bidirectional stream, the
  endpoint sends the outgoing Baton message on that stream.
* If the Baton message arrived on a self-initiated bidirectional stream, the
  endpoint opens a unidirectional stream and sends the outgoing Baton message on
  it.

If the endpoint has insufficient stream credit to open the correct type of
stream, it MUST close the WebTransport session with the DA_YAMN
session error code.

If the endpoint has insufficient flow control credit to send the Baton message,
it SHOULD send as much as limits allow, and wait for additional credit.  The
endpoint SHOULD close the WebTransport session with the BORED session error code
if the peer takes too long to grant credit.

## Baton message

~~~~~~~~~~ ascii-art
Baton Message {
  padding length(i)
  padding(...)
  baton(1)
}
~~~~~~~~~~

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

## Session Closure

To close a Devious Baton Session with an error, the endpoint
initiating the close sends a CLOSE_WEBTRANSPORT_SESSION capsule with
the specified session error code.  To close the session without an error, the
endpoint initiating the close sends a FIN on the CONNECT stream.

## Error Handling

If an endpoint receives a gracefully closed stream or datagram with an
incomplete Baton message, it MUST close the WebTransport session with the BRUH
session error code.

Either endpoint can send a STOP_SENDING or RESET_STREAM on an open stream.
STOP_SENDING MUST use the IDC stream error code. Upon receipt of a STOP_SENDING on a
stream, or a RESET_STREAM on a bidirectional stream, the endpoint MUST send a
RESET_STREAM for that stream with the WHATEVER stream error code unless it has already
closed the stream.  A RESET_STREAM sent spontaneously MUST use the I_LIED
stream error code.  If an endpoint detects that all baton streams have been reset, it
MUST close the WebTransport session with the GAME_OVER session error code.

If an endpoint gets tired of waiting for the next Baton message, it MAY close
the WebTransport session with the BORED error code.

### Stream Error Codes

The following error codes can be sent in RESET_STREAM and STOP_SENDING frames.

| Name      |  Code  | Description                    |
| --------- | :----: | ------------------------------ |
| IDC       |  0x01  | I don't care about this stream |
| WHATEVER  |  0x02  | The peer asked for this |
| I_LIED    |  0x03  | Spontaneous reset |
{: title="Stream Error Codes"}

### Session Error Codes

The following error codes can be sent in the CLOSE_WEBTRANSPORT_SESSION capsule.

| Name      |  Code  | Description                         |
| --------- | :----: | ----------------------------------- |
| DA_YAMN   |  0x01  | There is insufficient stream credit to continue the protocol |
| BRUH      |  0x02  | Received a malformed Baton message  |
| GAME_OVER |  0x03  | All baton streams have been reset   |
| BORED     |  0x04  | Got tired of waiting for the next message |
{: title="Session Error Codes"}

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

Error code naming inspiration by middle schoolers everywhere, but
specifically James Frindell.
