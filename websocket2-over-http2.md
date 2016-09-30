% Title = "WebSocket2 over HTTP/2"
% abbrev = "WebSocket2 over HTTP/2"
% category = "info"
% docName = "draft-svirid-websocket2-over-http2"
% ipr= "trust200902"
% area = "Internet"
% workgroup = "Network Working Group"
%
% date = 2016-10-01T00:00:00Z
%
% [[author]]
% initials="I."
% surname="Svirid"

.# Abstract

This document specifies a new protocol called WebSocket2 ontop of HTTP/2. 
The WebSocket2 protocol enables two-way binary communication 
between a client running untrusted code in a controlled environment to a 
remote host that has opted-in to communications from that code.  

This protocol has little incommon with the WebSocket protocol [@?RFC6455] other
than client side API compatibility.

Please send feedback to the ietf-http-wg@w3.org mailing list.

{mainmatter}

#  Introduction

You can read about why two way data streaming is important from the introduction
to WebSockets in RFC6455 Section 1 https://tools.ietf.org/html/rfc6455#section-1.

WebSocket2 over HTTP/2 is a protocol ontop HTTP/2 designed for modern times. Previous WebSocket client side API will be fully backward compatible with 
WebSocket2.

In this document, we describe WebSocket2 and how to layer WebSocket2 semantics onto HTTP/2 semantics by defining detailed mapping, replacement of
operations and events.

##  Conventions and Terminology

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**",
"**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this
document are to be interpreted as described in RFC 2119 [@!RFC2119].

Additionally, the key words "**MIGHT**", "**COULD**", "**MAY WISH TO**", "**WOULD
PROBABLY**", "**SHOULD CONSIDER**", and "**MUST (BUT WE KNOW YOU WON'T)**" in
this document are to interpreted as described in RFC 6919 [@!RFC6919].

#  Overview

WebSocket2 is functionally equivalent to binary streaming between a sandboxed 
client and inexplicit host, but layered on top of HTTP2.  Key advantages of 
WebSocket2 frames over HTTP/2 include:

   o  Two-way real time communication

   o  Server push and delivery without client request

   o  Binary transmission


## WebSocket2 Protocol

The protocol has two main parts, the handshake and data transfer. Data 
transmitted using WebSocket2 supports compression. 

The protocol relies on HTTP/2 for flow control.


#  Handshake

The main job of the handshake is to request authority and if granted by
the server, to negotiate a compression medium.

## Client Handshake Request
    
The client MUST send the path and the authority to the
server, the server is responsible for verifing the authority
and validating it.

The client MUST send a websocket2-version header that MUST specify
the websocket2 version being used.

The client MAY send a websocket2-compression header that advertises
the compression methods the client supports. Valid key value pairs include:

    o lz4=1-9;
        * This client supports lz4 with compression levels from 1 to 9

    o lz4=1,3,5-8;
        * This client supports lz4 with compression levels 1, 3 and 5-8

    o deflate=8-15;
        * This client supports deflate with sliding window bits from 8-15

Duplicate keys MUST NOT be present.

The client MUST NOT set the END_STREAM flag when sending the headers.

A client handshake request may look like:

~~~
:authority: example.org
:method: GET
:path: /ws2
websocket2-version: 1
websocket2-compression: lz4=1-9; deflate=8-15;
~~~

## Server Handshake Reply

The server MUST send a status 200 when accepting the peer or it 
MUST send a status 501 when rejecting the peer. END_STREAM on the HTTP/2 Data frame MUST only be set in the case of rejection.

The server MUST send ONLY ONE of the advertised compression methods
or exclude the websocket2-compression header from the reply, signaling
that no compression will be used.

A server handshake reply may look like:
~~~
:status: 200
websocket2-compression: lz4=1;
~~~
This signals that the server chose to use lz4 with a compression level of 1. 
Now both the client and server MUST use only this compression method.

## Invalid Requests

If a request is made to a WebSocket2 authority and path without the
websocket2-version header, a 501 :status MUST be replied.

If a request is made to a non WebSocket2 authority and path containing
the websocket2-version header, a 501 :status MUST be replied.

If the server rejects the client due to inability or unwillingness to 
negotiate, a 501 :status MUST be replied.

Any other error conditions are outside the scope of WebSocket2.

## Post Handshake

Following the handshake the client or server MUST NEVER set the
END_STREAM flag on any HTTP/2 DATA frame UNLESS the stream is to be
gracefully terminated.  Only a HTTP/2 DATA frame containing a WebSocket2 error frame allow the END_STREAM flag to be set.

# Data Framing

Once a handshake has been successfully completed the remote endpoints
can begin to send data to each other.  Data is sent using the HTTP/2 transport
layer fully adhering to DATA Frames, Secton 6.1 [@?RFC7540]. WebSocket2 has its own encapsulated framing protocol that is not to be confused with HTTP/2 DATA 
Frames.

Three frame types are defined:

: <br/>text   (represented by 0)
: <br/>binary (represented by 1)
: <br/>error  (represented by 2)

A WebSocket2 over HTTP/2 frame starts with 4 bits that are reserved that MUST 
be set to 0. 

The next 4 bits specify the frame type.

Next is a 32 bit unsigned integer in little endian format to specify the length
of the payload.  The maximum length of a payload is 4294967295.

Next is the payload which MAY be compressed if compression was negotiated. 

The term PAYLOAD in this section refers to data AFTER decompression if 
compression was negotiated.

~~~
    0         1        2        3       4        5             
+--------+--------+--------+--------+--------+--------+------+
| R - T  |                                   |               | ->
| S | Y  |            Payload Size           |    Payload    | ->
| V | P  |               (32)                |               | ->
|   - E  |                                   |               | ->
+--------+--------+--------+--------+--------+--------+------+
~~~

## Text Frame

The text frame has a Frame Type value of 0.  The payload must
be UTF-8 encoded text.  The whole message MUST contain valid UTF-8.
Invalid UTF-8 in the payload requires the remote end point to send
an error frame and close their side of the stream.

## Binary Frame

The binary frame has a Frame Type value of 1. The payload must be arbitrary
binary data which the application layer passes without comprehension.

## Error Frame

The error frame has a Frame Type value of 2.  It MUST contain a VALID 
4 byte error code.  If the code is shorter than 4 bytes, 0 bytes MUST
make up the rest to fill a total of 4 bytes.  Currently there are no
error codes less than 4 bytes.

The HTTP/2 transport layer DATA frame carrying the WebSocket2 error frame MUST
have the END_STREAM flag set.

No further WebSocket2 frames may be sent from this point onward and the stream is half closed.

The remote endpoint that receives the error frame MUST flush all pending sends
followed by an error frame of its own with the OKAY error code.  The HTTP/2 Data frame carrying this WebSocket2 frame MUST have the END_STREAM flag set.
~~~
    0         1        2        3        4                  
+--------+--------+--------+--------+--------+
|   -    |        Error Prefix Code          |      
| 0 | 2  |            (32 bits)              | 
|   -    |                                   |   
+--------+--------+--------+--------+--------+
~~~

### Error Frame Prefix Codes

  Valid error frame prefix codes currently are:
    
  OKAY
: <br/>* This should be sent went a client or server want to close the stream
    gracefully.

  UTF8
: <br/>* This should be sent when invalid UTF8 was passed in a text frame by a 
    remote endpoint.

  COMP
: <br/>* This should be sent when decompression failed by a remote endpoint.

  FRAM
: <br/>* This should be sent when an invalid Frame Type was specified.


{#fig-compression}
# Compression

WebSockets defined one compression method which used deflate and
kept a sliding window.  This compression is great for text like json but
pretty poor at everything else. Also keeping a sliding window is memory
intensive.

Lz4 is a compression method that is great for any kind of data while being very
cheap.  WebSocket2 is expected to transmit beyond text.

Currently two compression methods are defined lz4 and deflate.
The protocol is open to tweaking or accepting more in the future.  

## LZ4 Compressed Payload

A lz4 compressed payload is a 32 bit unsigned integer in little endian format
of the decompressed byte size followed by the actual compressed bytes.  The lz4 
compression level to use MUST be what was negotiated in the handshake.

## Deflate Compressed Payload

The deflate compression method builds from WebSocket but defines
more strict bounds.  There is no ability to reset compression context, if such
an option is required consider using lz4 with compression level 9 instead, 
it offers similar compression ratio and speed without the contexts memory overhead.
~~~
    * Both client and server MUST use the sliding window bits as determined by the server.
    * The client MUST use a memory level of 8.
    * The client MUST use a compression level of 9 (best_compression).
~~~
Any sliding window MUST NEVER have its context reset.

If the deflated payload trailing 4 bytes are 0x00, 0x00, 0xFF, 0xFF, remove
them before sending the payload.

Before inflating the payload append 0x00, 0x00, 0xFF, 0xFF to the end of it.


{backmatter}

# Acknowledgements

The author wishes to thank the participants of the WebSocket protocol,
participants of the HTTP/2 protocol and participants of the QUIC protocol.
