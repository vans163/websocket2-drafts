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


#  Handshake

The main job of the handshake is to request authority and if granted by
the server, to negotiate a compression medium.

## Client Handshake Request

The client MUST use the :method CONNECT.

The client MUST send a sec-ws2-version header that MUST specify
the websocket2 version being used.

The client MAY send a sec-ws2-compression header that advertises
the compression methods the client supports. Valid key value pairs include:

    o lz4=1-9;
        * This client supports lz4 with compression levels from 1 to 9

    o lz4=1;
        * This client supports lz4 with compression levels 1

    o deflate=8-15;
        * This client supports deflate with sliding window bits from 8-15

Duplicate keys MUST NOT be present.

The client MUST NOT set the END_STREAM flag when sending the headers.

A client handshake request may look like:

~~~
:method: CONNECT
:scheme: wss
:authority: example.org
:path: /ws2
sec-ws2-version: 1
sec-ws2-compression: lz4=1-9; deflate=8-15;
~~~

## Server Handshake Reply

END_STREAM on the HTTP/2 Header frame MUST only be set in the case of 
rejection.

The server MUST send ONLY ONE of the advertised compression methods
or exclude the sec-ws2-compression header from the response, signaling
that no compression will be used.

The server MUST include the sec-ws2-error header in the reply with an outlined
error reason.

A successful server handshake reply may look like:
~~~
:status: 200
sec-ws2-compression: lz4=1;
sec-ws2-error: success
~~~
This signals that the server chose to use lz4 with a compression level of 1. 
Now both the client and server MUST use only this compression method.

### Handshake Error Reasons

Valid error reasons are:

~~~
    success
        * The server accepted the client

    invalid_version
        * This version of websockets is not supported by the server

    cannot_negotiate_compression
        * This means the client did not offer any compression that 
        the server requires

    rejected
        * This means the server rejected the client and does not want
        to say why. This means the server supports websockets, but 
        rejected this particular client
~~~

## Post Handshake

Following the handshake the client or server MUST NEVER set the
END_STREAM flag on any HTTP/2 DATA frame UNLESS the stream is to be
gracefully terminated.  Only a HTTP/2 DATA frame containing a WebSocket2 error frame allow the END_STREAM flag to be set.

# Data Framing

Once a handshake has been successfully completed the remote endpoints
can begin to send data to each other.  Data is sent using the HTTP/2 transport
layer fully adhering to DATA Frames, Section 6.1 [@?RFC7540]. WebSocket2 has its own encapsulated framing protocol that is not to be confused with HTTP/2 DATA Frames.

Three frame types are defined:

: <br/>text   (represented by 0)
: <br/>binary (represented by 1)
: <br/>error  (represented by 2)

Three compression types are defined:

: <br/>none    (represented by 0)
: <br/>lz4     (represented by 1)
: <br/>deflate (represented by 2)

A WebSocket2 over HTTP/2 frame starts with the full frame length in a special 
format called VarSize [##VarSize]. If the first octet is less than 254, this 
is the full frame length. If the first octet is 254, read the next 16 bits as a 
little endian unsigned number for the frame length. If the first octet is 255, 
read the next 32 bits as a little endian unsigned number for the frame length.

Next are 4 reserved bits that MUST be set to 0. 

The next 2 bits specify the compression type.

The next 2 bits specify the frame type.

Next is the payload which MAY be compressed if compression was negotiated. 

The term PAYLOAD in this section refers to data AFTER decompression if 
compression was negotiated.

~~~
       0              1             2        
+--------------+--------------+--------------+
| Frame Length | R  - C - F |                | ->
|  (8,24,40)   | S  | T | T |   Payload      | ->
|              | V  | P | P |                | ->
|              |(4) -(2)-(2)|                | ->
+--------------+------------+----------------+
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
followed by an error frame of its own with the CLOS error code.  The HTTP/2 Data frame carrying this WebSocket2 frame MUST have the END_STREAM flag set.

The full websocket2 error frame with the length:

~~~
    0     01234567     2        3        4      5            
+--------+--------+--------+--------+--------+--------+
|        |        |         Error Code                |      
|   5    |00000010|          (32 bits)                | 
|        |        |                                   |   
+--------+--------+--------+--------+--------+--------+
~~~

### Error Frame Codes

  Valid error frame codes currently are:
    
  CLOS
: <br/>* This should be sent went a client or server want to close the stream
    gracefully.

  UTF8
: <br/>* This should be sent when invalid UTF8 was passed in a text frame by a 
    remote endpoint.

  COMP
: <br/>* This should be sent when decompression failed by a remote endpoint.

  FRAM
: <br/>* This should be sent when an invalid Frame Type was specified.

  LRGE
: <br/>* This should be sent when an endpoint rejects a frame due
    to it being too large.  This is up to the endpoint.

# VarSize

VarSize is a variable sized binary ranging from 1-5 octets.  It represents
an unsigned value.  If the first octet is equal to 254, read the next 16 bits 
as little endian unsigned.  If the first octet is equal
to 255, read the next 32 bits as little endian unsigned. Otherwise if the
first octet is less than 254, treat this as the final value.

{#fig-compression}
# Compression

WebSocket defined one compression method which used deflate and
kept a sliding window.  This compression is great but has limitations. Also 
keeping a sliding window is memory intensive.

Lz4 is a compression method that is great for any kind of data while being very
cheap.

Currently two compression methods are defined lz4 and deflate.
The protocol is open to tweaking or accepting more in the future.  

## LZ4 Compressed Payload

A lz4 compressed payload is a VarSize number of the decompressed byte size
followed by the actual compressed bytes. The lz4 compression level to use MUST 
be what was negotiated in the handshake.

A lz4 compressed payload may look like:

~~~
    0         1       2        3                   
+--------+--------+--------+--------+--------+--------+
|        |   Decompressed  |     Compressed payload   | ->     
|  254   |      Size       |                          | -> 
|        |                 |                          | ->  
+--------+--------+--------+--------+--------+--------+
~~~


## Deflate Compressed Payload

The deflate compression method implements [@!RFC7692] but defines
more strict bounds.  There is no ability to reset compression context.
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

The author wishes to thank Kari hurtta for contributing the handshake.

The author wishes to thank the participants of the WebSocket protocol,
participants of the HTTP/2 protocol and participants of the QUIC protocol.