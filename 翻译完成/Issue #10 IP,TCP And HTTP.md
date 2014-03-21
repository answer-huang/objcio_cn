---
layout: post
title: "IP, TCP, and HTTP"
category: "10"
date: "2014-03-07 06:00:00"
tags: article
author: "<a href=\"http://twitter.com/danielboedewadt\">Daniel Eggert</a>"
---

When an app communicates with a server, more often than not, that communication happens over HTTP. HTTP was developed for web browsers: when you enter http://www.objc.io into your browser, the browser talks to the server named www.objc.io using HTTP.

HTTP is an application protocol running at the application layer. There are several protocols layered on top of each other. The stack of layers is often depicted like this:


    Application Layer -- e.g. HTTP
    ----
    Transport Layer -- e.g. TCP
    ----
    Internet Layer -- e.g. IP
    ----
    Link Layer -- e.g. IEEE 802.2

The so-called [OSI (Open Systems Interconnection) model](https://en.wikipedia.org/wiki/OSI_model) defines seven layers. We'll take a look at the application, transport, and Internet layers for the typical HTTP usage: HTTP, TCP, and IP. The layers below IP are the data link and physical layers. These are the layers that, e.g. implement Ethernet (Ethernet has a data link part and a physical part).

We will only look at the application, transport, and Internet layers, and in fact only look at one particular combination: HTTP running on top of TCP, which in turn runs on top of IP. This is the typical setup most of us use for our apps, day in and day out.

We hope that this will give you a more detailed understanding of how HTTP works under the hood, as well as what some common problems are, and how you can avoid them.

There are other ways than HTTP to send data through the Internet. One reason that HTTP has become so popular is that it will almost always work, even when the machine is behind a firewall.

Let's start out at the lowest layer and take a look at IP, the Internet Protocol.

## IP — Internet Protocol

The **IP** in TCP/IP is short for [Internet Protocol](https://en.wikipedia.org/wiki/Internet_Protocol). As the name suggests, it is one of the fundamental protocols of the Internet.

IP implements [packet-switched networking](https://en.wikipedia.org/wiki/Packet_switching). It has a concept of *hosts* which are machines. The IP protocol specifies how *datagrams* (packets) are sent between these hosts.

A packet is a chunk of binary data that has a source host and a destination host. An IP network will then simply transmit the packet from the source to the destination. One important aspect of IP is that packets are delivered using *best effort*. A packet may be lost along the way and never reach the destination. Or it may get duplicated and arrive multiple times at the destination.

Each host in an IP network has an address -- the so-called *IP address*. Each packet contains the source and destination hosts' addresses. The IP is responsible for routing datagrams: as the IP packet travels through the network, each node (host) that it travels through looks at the destination address in the packet to figure out in which direction the packet should be forwarded.

Today, most packages are still IPv4 (Internet Protocol version 4), where each IPv4 address is 32 bits long. They're most often written in [dotted-decimal](https://en.wikipedia.org/wiki/Dotted_decimal) notation, like so: 198.51.100.42

The newer IPv6 standard is slowly gaining traction. It has an address space: its addresses are 128 bits long. This allows for easier routing as the packets travel through the network. And since there are more available addresses, tricks such as [network address translation](https://en.wikipedia.org/wiki/Network_address_translation) are no longer necessary. IPv6 addresses are represented in the hexadecimal system and divided into eight groups separated by colons, e.g. `2001:0db8:85a3:0042:1000:8a2e:0370:7334`.

### The IP Header

An IP packet consists of a header and a payload.

The payload contains the actual data to be transmitted, while the header is metadata.

#### IPv4 Header

An IPv4 header looks like this:

    IPv4 Header Format
    Offsets  Octet    0                       1                       2                       3
    Octet    Bit      0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31|
     0         0     |Version    |IHL        |DSCP            |ECN  |Total Length                                   |
     4        32     |Identification                                |Flags   |Fragment Offset                       |
     8        64     |Time To Live           |Protocol              |Header Checksum                                |
    12        96     |Source IP Address                                                                             |
    16       128     |Destination IP Address                                                                        |
    20       160     |Options (if IHL > 5)                                                                          |

The header is 20 bytes long (without options, which are rarely used).

The most interesting parts of the header are the source and destination IP addresses. Aside from that, the version field will be set to 4 -- it's IPv4. And the *protocol* field specified which protocol the payload is using. TCP's protocol number is 6. The total length field specified is the length of the entire packet -- header plus payload.

Check Wikipedia's [article on IPv4](https://en.wikipedia.org/wiki/IPv4_header) for all the details about the header and its fields.

#### IPv6 Header


IPv6 uses addresses that are 128 bits long. The IPv6 header looks like this:

    Offsets  Octet    0                       1                       2                       3
    Octet    Bit      0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31|
     0         0     |Version    |Traffic Class         |Flow Label                                                 |
     4        32     |Payload Length                                |Next Header            |Hop Limit              |
     8        64     |Source Address                                                                                |
    12        96     |                                                                                              |
    16       128     |                                                                                              |
    20       160     |                                                                                              |
    24       192     |Destination Address                                                                           |
    28       224     |                                                                                              |
    32       256     |                                                                                              |
    36       288     |                                                                                              |

The IPv6 header has a fixed length of 40 bytes. It's a lot simpler than IPv4 -- a few lessons were learned in the years that have passed since IPv4.

The source and destination addresses are again the most interesting fields. In IPv6 the *next header* field specifies what data follows the header. IPv6 allows chaining of headers inside the packet. Each subsequent IPv6 header will also have a *next header* field until the actual payload is reached. When the *next header* field, for example, is 6 (TCP's protocol number), the rest of the packet will be TCP data.

Again: Wikipedia's [article on IPv6 packets](https://en.wikipedia.org/wiki/IPv6_packet) has a lot more detail.

### Fragmentation

In IPv4, packets (datagrams) can get [fragmented](https://en.wikipedia.org/wiki/IP_fragmentation). The underlying transport layer will have an upper limit to the length of packet it can support. In IPv4, a router may fragment a packet if it gets routed onto an underlying data link for which the packet would otherwise be too big. These packets will then get reassembled at the destination host. 

In IPv6, a router will drop the packet and send back a *Packet Too Big* message to the sender. The end points use this to figure out what the so-called maximum transfer unit (MTU) is. Only when the minimum payload size is too big for that MTU will IPv6 use [fragmentation](https://en.wikipedia.org/wiki/IPv6_packet#Fragmentation).

## TCP — Transmission Control Protocol

One of the most common protocols to run on top of IP is, by far, TCP. It's so common that the entire suite of protocols is often referred to as TCP/IP.

The IP protocol allows for sending single packets (datagrams) between two hosts. Packets are delivered *best effort* and may: reach the destination in a different order than the one in which they were sent, reach the destination multiple times, or never reach the destination at all.

TCP is built on top of IP. The Transmission Control Protocol provides reliable, ordered, error-checked delivery of a stream of data between programs. With TCP, an application running on one device can send data to an application on another device and be sure that the data arrives there in the same way that it was sent. This may seem trivial, but it's really a stark contrast to how the raw IP layer works.

With TCP, applications establish connections between each other. A TCP connection is duplex and allows data to flow in both directions. The applications on either end do not have to worry about the data being split up into packets, or the fact that the packet transport is *best effort*. TCP guarantees that the data will arrive at the other end in pristine condition.

A typical use case of TCP is HTTP. Our web browser (application 1) connects to a web server (application 2). Once the connection is made, the browser can send a request through the connection, and the web server can send a response back through the same connection.

Multiple applications on the same host can use TCP simultaneously. To uniquely identify an application, TCP has a concept of *ports*. A connection between two applications has a source IP address and a source port on one end, and a destination IP address and a destination port at the other end. This pair of addresses, plus a port for either end, uniquely identifies the connection.

A web server using HTTPS will *listen* on port 443. The browser will use a so-called *ephemeral port* as the source port and then use TCP to establish a connection between the two address-port pairs.

TCP runs unmodified on top of both IPv4 and IPv6. The *Protocol* (IPv4) or *Next Header* (IPv6) field will be set to 6, which is the protocol number for TCP.

### TCP Segments

The data stream that flows between hosts is cut up into chunks, which are turned into TCP segments. The TPC segment then becomes the payload of an IP packet.

Each TCP segment has a header and a payload. The payload is the actual data chunk to be transmitted. The TCP segment header first and foremost contains the source and destination port number -- the source and destination addresses are already present in the IP header.

The header also contains sequence and acknowledgement numbers and quite a few other fields which are all used by TCP to manage the connection.

We'll go into more detail about sequence number in a bit. It's basically a mechanism to give each segment a unique number. The first segment has a random number, e.g. 1721092979, and subsequent segments increase this number by 1: 1721092980, 1721092981, and so on. The acknowledgement numbers allow the other end to communicate back to the sender regarding which segments it has received so far. Since TCP is duplex, this happens in both directions.

### TCP Connections

Connection management is a central component of TCP. The protocol needs to pull a lot of tricks to hide the complexities of the unreliable IP layer. We'll take a quick look at connection setup, the actual data flow, and connection termination.

The state transitions that a connection can go through are quite complex (c.f. [TCP state diagram](https://upload.wikimedia.org/wikipedia/commons/f/f6/Tcp_state_diagram_fixed_new.svg)). But in most cases, things are relatively simple.

#### Connection Setup

In TCP, a connection is always established from one host to another. Hence, there are two different roles in connection setup: one end (e.g. the web server) is listening for connections, while the other end (e.g. our app) connects to the listening application (e.g. the web server). The server performs a so-called *passive open* -- it starts listening. The client performs a so-called *active open* toward the server.

Connection setup happens through a three-way handshake. It works like this:

 1. The client sends a **SYN** to the server with a random sequence number, `A`
 2. The server replies with a **SYN-ACK** with an acknowledgment number of `A+1` and a random sequence number, `B`
 3. The client sends an **ACK** to the server with an acknowledgement number of `B+1` and a sequence number of `A+1`

**SYN** is short for *synchronize sequence numbers*. Once data flows between both ends, each TCP segment has a sequence number. This is how TCP makes sure that all parts arrive at the other end, and that they're put together in the right order. Before communication can start, both ends need to synchronize the sequence number of the first segments.

**ACK** is short for *acknowledgment*. When a segment arrives at one of the ends, that end will acknowledge the receipt of that segment by sending an acknowledgment for the sequence number of the received segment.

If we run´:

    curl -4 http://www.apple.com/contact/

this will cause `curl` to create a TCP connection to www.apple.com on port 80.

The server www.apple.com / 23.63.125.15 is listening on port 80. Our own address in the output is `10.0.1.6`, and our *ephemeral port* is `52181` (this is a random, available port). The output from `tcpdump(1)` for the three-way handshake looks like this:

    % sudo tcpdump -c 3 -i en3 -nS host 23.63.125.15
    18:31:29.140787 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [S], seq 1721092979, win 65535, options [mss 1460,nop,wscale 4,nop,nop,TS val 743929763 ecr 0,sackOK,eol], length 0
    18:31:29.150866 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [S.], seq 673593777, ack 1721092980, win 14480, options [mss 1460,sackOK,TS val 1433256622 ecr 743929763,nop,wscale 1], length 0
    18:31:29.150908 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [.], ack 673593778, win 8235, options [nop,nop,TS val 743929773 ecr 1433256622], length 0

That's a lot of information right there. Let's step through this bit by bit.

On the very left-hand side we see the system time. This was run in the evening at 18:31. Next, `IP` tells us that these are IP packets.

Next we see `10.0.1.6.52181 > 23.63.125.15.80`. This is the source and destination address-port pair. The first and third lines are from the client to the server, the second from the server to the client. `tcpdump` will simply append the port number to the IP address. `10.0.1.6.52181` means IP address 10.0.1.6, port 52181.

The `Flags` are flags in the TCP segment header: `S` for **SYN**, `.` for **ACK**, `P` for **PUSH**, and `F` for **FIN**. There are a few more we won't see here. Note how these three lines have **SYN**, then **SYN-ACK**, then **ACK**: this is the three-way handshake.

The first line shows the client sending the random sequence number 1721092979 (A) to the server. The second line shows the server sending an acknowledgement for 1721092980 (A+1) and its random sequence number 673593777 (B). Finally, the third line shows the client acknowledging 673593778 (B+1).

#### Options

Another thing that happens during connection setup is for both ends to exchange additional options. In the first line, we see the client sending:

    [mss 1460,nop,wscale 4,nop,nop,TS val 743929763 ecr 0,sackOK,eol]

and on the second line, the server is sending:

    [mss 1460,sackOK,TS val 1433256622 ecr 743929763,nop,wscale 1]

The `TS val` / `ecr` are used by TCP to estimate the round-trip time (RTT). The `TS val` part is the *time stamp* of the sender, and the (`ecr`) is the timestamp *echo reply*, which is (generally) the last timestamp that the sender has received so far. TCP uses the round-trip time for its congestion-control algorithms.

Both ends are sending `sackOK`. This will enable *Selective Acknowledgement*. It switches the sequence numbers and acknowledgment number to use byte range instead of TCP segment numbers. And it then allows both ends to acknowledge receipt by byte range. This is outlined in [section 3 of RFC 2018](http://tools.ietf.org/html/rfc2018#section-3).

The `mss` option specified the *Maximum Segment Size*, which is the maximum number of bytes that this end is willing to receive in a single segment. `wscale` is the *window scale factor* that we'll talk about in a bit.

#### Connection Data Flow

When the connection is created, both ends can send data to the other end. Each segment that is sent has a sequence number one larger than the previous segment. The receiving end will acknowledge packets as they are received by sending back segments with the corresponding **ACK**.

In theory this may looks like:

    host A sends segment with seq 0
    host A sends segment with seq 1
    host A sends segment with seq 2     host B sends segment with ack 0
    host A sends segment with seq 3     host B sends segment with ack 1
                                        host B sends segment with ack 2
                                        host B sends segment with ack 3

This mechanism happens in both directions. Host A will keep sending packets. As they arrive at host B, host B will send acknowledgements for these packets back to host A. But host A will keep sending packets without waiting for host B to acknowledge them.

TCP incorporates flow control and a lot of sophisticated mechanisms for congestion control. These are all about figuring out (A) if segments got lost and need to be retransmitted, and (B) if the rate at which segments are sent needs to be adjusted.

Flow control is about making sure that the sending side doesn't send data faster than the receiver (at either end) can process it. The receiver sends a so-called *receive window*, which tells the sender how much more data the receiver can buffer. There are some subtle details we'll skip, but in the above `tcpdump` output, we see a `win 65535` and a `wscale 4`. The first is the window size, the latter a scale factor. As a result, the host at `10.0.1.6` says it has a receive window of 4·64 kB = 256 kB and the the host at `23.63.125.15` advertises `win 14480` and `wscale 1` i.e. roughly 14 kB. As either side receives data, it will send an updated receive window to the other end.

Congestion control is quite complex. The various mechanisms are all about figuring out at which rate data can be sent through the network. It's a very delicate balance. On one hand, there's the obvious desire to send data as fast as possible, but on the other hand, the performance will collapse dramatically when sending too much data. This is called [congestive collapse](https://en.wikipedia.org/wiki/Congestive_collapse#Congestive_collapse) and it's a property inherent to packet-switched networks. When too many packets are sent, packets will collide with other packets and the packet loss rate will climb dramatically.

The congestion control mechanisms need to also make sure that they play well with other flows. Today's congestion control mechanisms in TCP are outlined in detail in about 6,000 words in [RFC 5681](https://www.rfc-editor.org/rfc/rfc5681.txt). The basic idea is that the sender side looks at the acknowledgments it gets back. This is a very tricky business and there are various tradeoffs to be made. Remember that the underlying IP packets can arrive out of order, not at all, or twice. The sender needs to estimate what the round-trip time is and use that to figure out if it should have received an acknowledgement already. Retransmitting packets is obviously costly, but not retransmitting causes the connection to stall for a while, and the load on the network is likely to be very dynamic. The TCP algorithms need to constantly adapt to the current situation.

The important thing to note is that a TCP connection is a very lively and flexible thing. Aside from the actual data flow, both ends constantly send hints and updates back and forth to continuously fine-tune the connection.

Because of this tuning, TCP connections that are short-lived can be very inefficient. When a connection is first created, the TCP algorithms still don't know what the conditions of the network are. And toward the end of the connection lifetime, there's less information flowing back to the sender, which therefore has a harder time estimating how things are moving along.

Above, we saw the first three segments between the client and the server. If we look at the remainder of the connection, it looks like this:

    18:31:29.150955 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [P.], seq 1721092980:1721093065, ack 673593778, win 8235, options [nop,nop,TS val 743929773 ecr 1433256622], length 85
    18:31:29.161213 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], ack 1721093065, win 7240, options [nop,nop,TS val 1433256633 ecr 743929773], length 0

The client at `10.0.1.6` sends the first segment with data `length 85` (the HTTP request, 85 bytes). The **ACK** number is left at the same value, because no data has been received from the other end since the last segment.

The server at `23.63.125.15` then acknowledges the receipt of that data (but doesn't send any data): `length 0`. Since the connection is using *Selective acknowledgments*, the sequence number and acknowledgment numbers are byte ranges: 1721092980 to 1721093065 is 85 bytes. When the other end sends `ack 1721093065`, that means: I have everything up to byte 1721093065.

This continues until all data has been sent:

    18:31:29.189335 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673593778:673595226, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.190280 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673595226:673596674, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.190350 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [.], ack 673596674, win 8101, options [nop,nop,TS val 743929811 ecr 1433256660], length 0
    18:31:29.190597 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673596674:673598122, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.190601 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673598122:673599570, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.190614 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673599570:673601018, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.190616 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673601018:673602466, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.190617 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673602466:673603914, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.190619 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673603914:673605362, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.190621 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673605362:673606810, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.190679 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [.], ack 673599570, win 8011, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
    18:31:29.190683 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [.], ack 673602466, win 7830, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
    18:31:29.190688 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [.], ack 673605362, win 7830, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
    18:31:29.190703 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [.], ack 673605362, win 8192, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
    18:31:29.190743 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [.], ack 673606810, win 8192, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
    18:31:29.190870 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [.], seq 673606810:673608258, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
    18:31:29.198582 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [P.], seq 673608258:673608401, ack 1721093065, win 7240, options [nop,nop,TS val 1433256670 ecr 743929811], length 143
    18:31:29.198672 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [.], ack 673608401, win 8183, options [nop,nop,TS val 743929819 ecr 1433256660], length 0

#### Connection Termination

Finally the connection is torn down (or terminated). Each end will send a **FIN** flag to the other end to tell it that it's done sending. This **FIN** is then acknowledged. When both ends have sent their **FIN** flags and they have been acknowledged, the connection is fully closed:

    18:31:29.199029 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [F.], seq 1721093065, ack 673608401, win 8192, options [nop,nop,TS val 743929819 ecr 1433256660], length 0
    18:31:29.208416 IP 23.63.125.15.80 > 10.0.1.6.52181: Flags [F.], seq 673608401, ack 1721093066, win 7240, options [nop,nop,TS val 1433256680 ecr 743929819], length 0
    18:31:29.208493 IP 10.0.1.6.52181 > 23.63.125.15.80: Flags [.], ack 673608402, win 8192, options [nop,nop,TS val 743929828 ecr 1433256680], length 0

Note how on the second line, `23.63.125.15` sends its **FIN**, and at the same time acknowledges the other end's **FIN** with an **ACK** (dot), all in a single segment.

## HTTP — Hypertext Transfer Protocol

The [World Wide Web](https://en.wikipedia.org/wiki/World_Wide_Web) of interlinked hypertext documents and a browser to browse this web started as an idea set forward in 1989 at [CERN](https://en.wikipedia.org/wiki/CERN). The protocol to be used for data communication was the *Hypertext Transfer Protocol*, or HTTP. Today's version is *HTTP/1.1*, which is defined in [RFC 2616](https://tools.ietf.org/html/rfc2616).

### Request and Response

HTTP uses a simple request and response mechanism. When we enter http://www.apple.com/ into Safari, it send an HTTP request to the server at `www.apple.com`. The server in turn sends back a (single) response which contains the document that was requested.

There's always a single request and a single response. And both requests and responses follow the same format. The first line is the *request line* (request) or *status line* (response). This line is followed by headers. The headers end with an empty line. After that, an empty line follows the optional message body.

### A Simple Request

When [Safari](https://en.wikipedia.org/wiki/Safari_%28web_browser%29) loads the HTML for http://www.objc.io/about.html, it sends an HTTP request to `www.objc.io` with the following content:

    GET /about.html HTTP/1.1
    Host: www.objc.io
    Accept-Encoding: gzip, deflate
    Connection: keep-alive
    If-None-Match: "a54907f38b306fe3ae4f32c003ddd507"
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    If-Modified-Since: Mon, 10 Feb 2014 18:08:48 GMT
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_2) AppleWebKit/537.74.9 (KHTML, like Gecko) Version/7.0.2 Safari/537.74.9
    Referer: http://www.objc.io/
    DNT: 1
    Accept-Language: en-us


The first line is the **request line**. It specifies the desired action: `GET`. This is called the [HTTP method](https://en.wikipedia.org/wiki/HTTP_method#Request_methods). Next up the path of the resources that this action is supposed to be made on: `/about.html`, and finally the HTTP version. In this case, we want to *get* the document.

Next, we have 10 lines with 10 HTTP headers. These are followed by an empty line. There's no message body in this request.

The headers have very varying purposes. They convey additional information to the web server. Wikipedia has a nice list of [common HTTP header fields](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields). The first `Host: www.objc.io` header tells the server what server name the request is meant for. This mandatory request header allows the same physical server to serve multiple [domain names](https://en.wikipedia.org/wiki/Domain_names).

Let's look at a few common ones:

    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-us

This tells the server what media type Safari would like to receive. A server may be able to send a response in various formats. The `text/html` strings are [Internet media types](https://en.wikipedia.org/wiki/Mime_type), sometimes also known as MIME types or Content-types. The `q=0.9` allows Safari to convey a quality factor which it associates with the given media types. `Accept-Language` tells the server which languages Safari would prefer. Again, this lets the server pick the matching languages, if available.

    Accept-Encoding: gzip, deflate

With this header, Safari tells the server that the response body can be sent compressed. If this header is not set, the server must send the data uncompressed. Particularly for text (such as HTML), compression rates can dramatically reduce the amount of data that has to be sent.

    If-Modified-Since: Mon, 10 Feb 2014 18:08:48 GMT
    If-None-Match: "a54907f38b306fe3ae4f32c003ddd507"

These two are due to the fact that Safari already has the resulting document in its cache. Safari tells the server only to send it if it has either changed since February 10, or if its so-called ETag doesn't match `a54907f38b306fe3ae4f32c003ddd507`.

The `User-Agent` header tells the server what kind of client is making the request.

### A Simple Response

In response to the above, the server responds with:

    HTTP/1.1 304 Not Modified
    Connection: keep-alive
    Date: Mon, 03 Mar 2014 21:09:45 GMT
    Cache-Control: max-age=3600
    ETag: "a54907f38b306fe3ae4f32c003ddd507"
    Last-Modified: Mon, 10 Feb 2014 18:08:48 GMT
    Age: 6
    X-Cache: Hit from cloudfront
    Via: 1.1 eb67cb25620df959ba21a943fbc49ef6.cloudfront.net (CloudFront)
    X-Amz-Cf-Id: dDSBgR86EKBemW6el-pBI9kAnuYJEaPQYEqGmBnilD12CbixCuZYVQ==

The first line is the so-called *status line*. It contains the HTTP version, followed by a [status code](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes) (304) and a status message.

HTTP defines a [list of status codes](https://en.wikipedia.org/wiki/Http_status_codes) and their meanings. In this case, we're receiving **304**, which means that the resource we requested hasn't been modified.

The response doesn't contain any body message. It simply tells the receiver: *Your version is up to date.*

### Caching Turned Off

Let's do another request with `curl`:

    % curl http://www.apple.com/hotnews/ > /dev/null

`curl` doesn't use a local cache. The entire request will now look like this:

    GET /hotnews/ HTTP/1.1
    User-Agent: curl/7.30.0
    Host: www.apple.com
    Accept: */*

This is quite similar to what Safari was requesting. This time, there's no `If-None-Match` header and the server will have to send the document.

Note how `curl` announces that it will accept any kind of media format: (`Accept: */*`).

The response from www.apple.com looks like this:

    HTTP/1.1 200 OK
    Server: Apache
    Content-Type: text/html; charset=UTF-8
    Cache-Control: max-age=424
    Expires: Mon, 03 Mar 2014 21:57:55 GMT
    Date: Mon, 03 Mar 2014 21:50:51 GMT
    Content-Length: 12342
    Connection: keep-alive

    <!DOCTYPE html>
    <html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en-US" lang="en-US">
    <head>
        <meta charset="utf-8" />

and continues on for quite a while. It now has a response body which contains the HTML document.

The response from Apple's server contains a [status code](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes) of *200*, which is the standard response for successful HTTP requests.

Apple's server also tells us that the response's media type is `text/html; charset=UTF-8`. The `Content-Length: 12342` tells us what the length of the message body is.

## HTTPS — HTTP Secure

[Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security) is a cryptographic protocol that runs on top of TCP. It allows for two things: both ends can verify the identity of the other end, and the data sent between both ends is encrypted. Using HTTP on top of TLS gives you HTTP Secure, or simply, HTTPS.

Simply using HTTPS instead of HTTP will give you a huge improvement in security. There are some additional steps that you may want to take, though, both of which will additionally improve the security of your communication.

### TSL 1.2

You should set the `TLSMinimumSupportedProtocol` to `kTLSProtocol12` to require [TLS version 1.2](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_1.2) if your server supports that. This will make [man-in-the-middle attacks](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) more difficult.

### Certificate Pinning

There's little point in the data we send being encrypted if we can't be certain that the other end we're talking to is actually who we think it is. The server's certificate tells us who the server is. Only allowing a connection to a very specific certificate is called [certificate pinning](https://en.wikipedia.org/wiki/Certificate_pinning#Certificate_pinning).

When a client makes a connection over TLS to a server, the operating system will decide if it thinks the server's certificate is valid. There are a few ways this can be circumvented, most notably by installing certificates onto the iOS device and marking them as trusted. Once that's been done, it's trivial to perform a [man-in-the-middle attack](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) against your app.

To prevent this from happening (or at least make it extremely difficult), we can use a method called certificate pinning. When then TLS connection is set up, we inspect the server's certificate and check not only that it's to be considered valid, but also that it is the certificate that we expect our server to have. This only works if we're connecting to our own server and hence can coordinate an update to the server's certificate with an update to the app.

To do this, we need to inspect the so-called *server trust* during connection. When an `NSURLSession` creates a connection, the delegate receives a `-URLSession:didReceiveChallenge:completionHandler:` call. The passed `NSURLAuthenticationChallenge` has a `protectionSpace` property, which is an instance of `NSURLProtectionSpace`. This, in turn, has a `serverTrust` property.

The `serverTrust` is a `SecTrustRef` object. The Security framework has various methods to query the `SecTrustRef` object. The [`AFSecurityPolicy`](https://github.com/AFNetworking/AFNetworking/blob/7f2c395ba185b586468557b22977ccd2b79fae66/AFNetworking/AFSecurityPolicy.m) from the AFNetworking project is a good starting point. As always, when you build your own security-related code, have someone review it carefully. You don't want to have a [`goto fail;`](https://www.imperialviolet.org/2014/02/22/applebug.html) bug in this part of your code.

## Putting the Pieces Together

Now that we know a bit about how all the pieces (IP, TCP, and HTTP) work, there are a few things we can do and be aware of.


### Efficiently Using Connections

There are two aspects of a TCP connection that are problematic: the initial setup, and the last few segments that are pushed across the connection.

#### Setup

The connection setup can be very time consuming. As mentioned above, TCP needs to do a three-way handshake. There's not a lot of data that needs to be sent back and forth. But, particularly when on a mobile network, the time a packet takes to travel from one host (a user's iPhone) to another host (a web server) can easily be around 250 ms -- a quarter of a second. For a three-way handshake, we'll often spend 750 ms just to establish the connection, before any payload data has been sent.

In the case of HTTPS, things are even more dramatic. With HTTPS we have HTTP running on top of TLS, which in turn runs on top of TCP. The TCP connection will still do its three-way handshake. Next up, the TLS layer does another three-way handshake. Roughly speaking, an HTTPS connection will thus take twice as long as a normal HTTP connection before sending any data . If the round-trip time is 500 ms (250 ms end-to-end), that adds up to 1.5 seconds.

This setup time is costly, regardless of if the connection will transfer a lot or just a small amount of data.

Another aspect of TCP affects connections on which we expect to transfer a larger amount of data. When sending segments into a network with unknown conditions, TCP needs to probe the network to determine the available capacity. In other words: it takes TCP a while to figure out how fast it can send data over the network. Only once it has figured this out can it send the data at the optimal speed. The algorithm that is used for this is called [slow-start](https://en.wikipedia.org/wiki/Slow-start). On a side note, it's worth pointing out that the slow-start algorithm doesn't perform well on networks with poor data link layer transmission quality, as is often the case for wireless networks.

#### Tear Down

Another problem arises toward the end of the data transfer. When we do an HTTP request for a resource, the server will keep sending TCP segments to our host, and the host will respond with **ACK** messages as it receives the data. If a single packet is lost along the way, the server will not receive an **ACK** for that packet. It can therefore notice that the packet was lost and do what's called a [fast retransmit](https://en.wikipedia.org/wiki/Fast_retransmit).

When a packet is lost, the following packet will cause the receiver to send an **ACK** identical to the last **ACK** it sent. The receiver will hence receive a *duplicate ACK*. There are several network conditions that can cause a duplicate ACK even without packet loss. The sender therefore only performs a fast retransmit when it receives three duplicate ACKs.

The problem with this is at the end of the data transmission. When the sender stops sending segments, the receiver stops sending ACKs back. And there's no way for the fast-retransmit algorithm to detect if a packet is lost within the last four segments being sent. On a typical network, that's equivalent to 5.7 kB of data. Within the last 5.7 kB, the fast retransmit can't do its job. If a packet is lost, TCP has to fall back to a more patient algorithm to detect packet loss. It's not uncommon for a retransmit to take several seconds in such a case.

#### Keep-Alive and Pipelining

HTTP has two strategies to counter these problems. The simplest is called [HTTP persistent connection](https://en.wikipedia.org/wiki/HTTP_persistent_connection), sometimes also known as *keep-alive*. HTTP will simply reuse the same TCP connection once a single request-response pair is done. In case of HTTPS, the same TLS connection will be re-used:

    open connection
    client sends HTTP request 1 ->
                                <- server sends HTTP response 1
    client sends HTTP request 2 ->
                                <- server sends HTTP response 2
    client sends HTTP request 3 ->
                                <- server sends HTTP response 3
    close connection

The second step is to use [HTTP pipelining](https://en.wikipedia.org/wiki/Http_pipelining), which allows the client to send multiple requests over the same connection without waiting for the response from previous requests. The requests can be sent in parallel with responses being received, but the responses are still sent back to the client in the order that the requests were sent -- it's still a [first-in-first-out](https://en.wikipedia.org/wiki/FIFO) principle. 

Slightly simplified, this would look like:

    open connection
    client sends HTTP request 1 ->
    client sends HTTP request 2 ->
    client sends HTTP request 3 ->
    client sends HTTP request 4 ->
                                <- server sends HTTP response 1
                                <- server sends HTTP response 2
                                <- server sends HTTP response 3
                                <- server sends HTTP response 4
    close connection

Note, however, that the server may start sending the response at any time, and doesn't have to wait until all requests have been received.

With this, we're able to use TCP in a more efficient way. We're now only doing the handshake at the very beginning, and since we keep using the same connection, TCP can do a much better job of using the available bandwidth. The TCP congestion control algorithms will also do a much better job. Finally, the problem with the fast retransmit failing to work on the last four segments will only affect the last four segments of the entire connection, and not the last four segments of each request-response, as before.

The improvements of using HTTP pipelining can be quite dramatic over high-latency connections -- which is what you have when your iPhone is not on Wi-Fi. In fact, there's been some [research](http://research.microsoft.com/pubs/170059/A%20comparison%20of%20SPDY%20and%20HTTP%20performance.pdf) that suggests that there's no additional performance benefit to using [SPDY](https://en.wikipedia.org/wiki/SPDY) over HTTP pipelining on mobile networks.

When communication with a server, [RFC 2616](https://tools.ietf.org/html/rfc2616) recommends using two connections when HTTP pipelining is enabled. According to these guidelines, this will result in the best response times and avoid congestion. There's no performance benefit to additional connections.

Sadly, there are still some servers that don't support pipelining. You should try to enable it, however, and check if your particular server does support it. Chances are, it does. But because some servers don't, HTTP pipelining is turned off by default for `NSURLSession`. Make sure to set `HTTPShouldUsePipelining` to `YES` on the `NSURLSessionConfiguration` if you're able to use pipelining.

### Timeout

We have all used apps while being on a slow network. Many of these apps give up after a timeout, say 15 seconds. That is really bad app design. It's probably good to give the user feedback and say, "Hey, your network seems to be slow. This may take some time." But as long as there's a connection, even if it's bad, TCP ensure that the request will make it to the other end and that the response will make it back. It may just take a while.

Or look at it another way: If we're on a slow network, the request-response round-trip time may take 17 seconds to complete. If we're stopping after 15 seconds, the user will be unable to perform the desired action if he or she is patient. If the user is on a slow network, he or she will know that things will take a while (we'll send a notification), and if the user is willing to be patient, we shouldn't prevent him or her from using the app.

There's a misconception that restarting the (HTTP) request will fix the problem. That is not the case. Again, TCP will resend those packets that need resending on its own.

The right approach is this: When we send off a URL request, we set a timer for 10 seconds. Once we get the response back, we invalidate the timer. If the timer fires before we get the response back, we'll show some UI: "Please be patient, the network appears to be slow right now." Depending on the app, we might want to give the user an option to cancel the action. But we shouldn't make that decision for the user.

### Caching

Note how in our first example, we send a:

    If-None-Match: "a54907f38b306fe3ae4f32c003ddd507"

line to the server, letting it know that we already have a resource locally and only want the server to send the resource to us if it has a newer version. If you're building your own communication with your server, try to make use of this mechanism. It can dramatically speed up communication if used correctly. The mechanism is called [HTTP ETag](https://en.wikipedia.org/wiki/HTTP_ETag).

Going to extremes, remember: "The fastest request is the one never made." When sending a request to a server, even on a very good network, requesting a small amount of data, and a very fast server, you're unlikely to have a response back in less than 50 ms. And that's just one request. If there's a way for you to create that data locally in 50 ms or less, don't make that request.

Cache resources locally whenever you think that they'll stay valid for some time. Check the `Expires` header or rely on the caching mechanisms of `NSURLSession` by using the default `NSURLRequestUseProtocolCachePolicy`.


## Summary

Sending a request over HTTP with `NSURLSession` is very straightforward. But there are a lot of technologies involved in making that request. Knowing how those pieces work will allow you to optimize the way you're using HTTP requests. Our users expect apps that perform well under any condition. Understanding how IP, TCP, and HTTP work makes it easier to deliver on those expectations.




<link type="text/css" media="screen" href="http://www.objc.io/css/fonts.css" rel="stylesheet"></link>

<link media="screen" type="text/css" rel="stylesheet" href="http://www.objc.io/assets/global-29d667822eb607e2d075178d32aee1d3.css"></link>

#IP,TCP,and HTTP
[Issue #10 Syncing Data](http://www.objc.io/issue-10/index.html),March 2014
[By Daniel Eggert](http://twitter.com/danielboedewadt)

当app和服务端进行通信的时候，大多数情况下，都是采用HTTP协议。HTTP最初是为web浏览器而定制的，如果在浏览器里输入http://www.objc.io，浏览器会通过HTTP协议和www.objc.io所对应的服务器进行通信。

HTTP是运行在应用层上的应用协议，而不同的层级上都有相应的协议在运行。层级的堆栈关系一般可以这么描述：

<pre><code class="objectivec">Application Layer -- e<span class="variable">.g</span>. HTTP
 - - - - 
Transport Layer -- e<span class="variable">.g</span>. TCP
 - - - - 
Internet Layer -- e<span class="variable">.g</span>. IP
 - - - -
Link Layer -- e<span class="variable">.g</span>. IEEE <span class="number">802.2</span></code></pre>

这就是[OSI（Open Systems Interconnection，开发式系统互联）](https://en.wikipedia.org/wiki/OSI_model)模型所定义的七层结构。为了更清楚的描述典型的HTTP的应用：HTTP,TCP以及IP，本文会重点关注一下应用层(application layer)、传输层(transport layer)和网络层(internet layer)的实现以及他们是如何支撑HTTP服务的。

如上文所述，只关注应用层，传输层和互联网模型层的部分，更确切的说，着重探讨一种特殊的混合模式：基于IP的TCP，以及基于TCP实现的HTTP。这就是我们每天使用的app的基本网络配置。

通过本文，希望大家能够对HTTP工作原理有一个细致的了解，知道一些常见的HTTP问题的产生原因，从而能在实践中尽量避免这些问题的发生。

其实在互联网上传递数据的方式并不只HTTP一种。HTTP之所以被广泛使用的原因是其非常稳定、易用，即便是防火墙一般也是允许HTTP协议穿透的。

接下来我们从最低的一层谈起，说说IP网络协议。

##IP-Internet Proctocol (IP网络协议)

TCP/IP中的IP是[Internet Protocol](https://en.wikipedia.org/wiki/Internet_Protocol)网络协议的缩写。从字面意思便知，它是互联网众多协议的基础。

IP采用[分组交换](https://en.wikipedia.org/wiki/Packet_switching)。同时IP协议明确了主机之间的数据报（数据包）的传输方式。

所谓数据包是指一段二进制数据，其中包含了发送源主机和目标主机的信息。IP网络负责源与目标主机之间的数据包传输。IP协议的特点是*best effort*（尽力服务，其目标是提供有效服务并尽力传输）。这意味着，在传输过程中，数据包可能会丢失，也有可能被重复传送导致目标主机收到多个同样的数据包。

IP网络中的主机都配有自己的地址，被称为IP地址。每个数据包中都包含了源主机和目标主机的IP地址。IP协议负责路由计算，即IP数据包在网络中的传输路径，数据包所经过的每一个主机节点都会读取数据包中的目标主机地址信息，以便选择朝什么地方传送数据包。

今天，绝大多数的数据包仍旧是IPv4（Internet Protocol version 4 网际协议版本4），每一个IPv4地址是长度为32位。常见采用[dotted-decimal](https://en.wikipedia.org/wiki/Dotted_decimal)（点分十进制）表示法，具体形式如：198.51.100.42。

新的IPv6标准也正在逐渐推广中。它有更大的地址空间：长度为128位。提高了路由中数据包的传输速度。另外，由于有更多的地址可以分配，诸如[网络地址转换](https://en.wikipedia.org/wiki/Network_address_translation)等问题也迎刃而解。IPv6的表示形式为：八组十六进制数以冒号分割，比如：2001:0db8:85a3:0042:1000:8a2e:0370:7334。

##IP Hearder (IP报头)

一个IP数据包通常包含报头信息和payload（有效载荷）。

payload中的内容即是要传输的真正信息，而报头header承载的是与传输数据有关的metadata（元数据）。

###IPv4 Header (IPv4 报头)

IPv4的报头信息内容如下：

<pre><code class="objectivec">IPv4 Header Format
Offsets  Octet    <span class="number">0</span>                       <span class="number">1</span>                       <span class="number">2</span>                       <span class="number">3</span>
Octet    Bit      <span class="number">0</span>  <span class="number">1</span>  <span class="number">2</span>  <span class="number">3</span>  <span class="number">4</span>  <span class="number">5</span>  <span class="number">6</span>  <span class="number">7</span>  <span class="number">8</span>  <span class="number">9</span> <span class="number">10</span> <span class="number">11</span> <span class="number">12</span> <span class="number">13</span> <span class="number">14</span> <span class="number">15</span> <span class="number">16</span> <span class="number">17</span> <span class="number">18</span> <span class="number">19</span> <span class="number">20</span> <span class="number">21</span> <span class="number">22</span> <span class="number">23</span> <span class="number">24</span> <span class="number">25</span> <span class="number">26</span> <span class="number">27</span> <span class="number">28</span> <span class="number">29</span> <span class="number">30</span> <span class="number">31</span>|
 <span class="number">0</span>         <span class="number">0</span>     |Version    |IHL        |DSCP            |ECN  |Total Length                                   |
 <span class="number">4</span>        <span class="number">32</span>     |Identification                                |Flags   |Fragment Offset                       |
 <span class="number">8</span>        <span class="number">64</span>     |Time To Live           |Protocol              |Header Checksum                                |
<span class="number">12</span>        <span class="number">96</span>     |Source IP Address                                                                             |
<span class="number">16</span>       <span class="number">128</span>     |Destination IP Address                                                                        |
<span class="number">20</span>       <span class="number">160</span>     |Options (<span class="keyword">if</span> IHL &gt; <span class="number">5</span>)                                                                          |</code></pre>

报文头长度为20字节（不包含极少用到的option可选项信息）。

报文信息中最关键的是源和目标IP地址。除此之外，版本信息(version)是4，代表IPv4。*protocol*（协议区）代表payload采用的传输协议。TCP的协议号是6。Total Length（总长度区）标明了header加payload整个数据包的大小。

详情参看维基百科中关于[IPv4的条目](https://en.wikipedia.org/wiki/IPv4_header)，里面有关于header各个区域信息的详细介绍。

###IPv6 Header (IPv6报头)

IPv6的地址长度为128位。IPv6的报头信息内容如下：

<pre><code class="undefined">Offsets  Octet    0                       1                       2                       3
Octet    Bit      0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31|
 0         0     |Version    |Traffic Class         |Flow Label                                                 |
 4        32     |Payload Length                                |Next Header            |Hop Limit              |
 8        64     |Source Address                                                                                |
12        96     |                                                                                              |
16       128     |                                                                                              |
20       160     |                                                                                              |
24       192     |Destination Address                                                                           |
28       224     |                                                                                              |
32       256     |                                                                                              |
36       288     |                                                                                              |</code></pre>

IPv6报头采用固定长度40字节。经过多年来对IPv4使用的总结，如今IPv6的报头信息简化了许多。

除了源和目标地址这种必备信息外。IPv6提供专门的*next header*（下一个头信息）区域用于存放扩展头信息，下一个头字段值指明是否有下一个扩展头及下一个扩展头是什么，因此，IPv6头可以链接起来，从基本的IPv6头开始，逐个链接各扩展头。在没有扩展头的IPv6包中，此字段的值表示上一层协议，如果是6就表示TCP协议，数据包的其他信息就是TCP协议要传输的数据。

同样的，更多信息请参考维基百科上关于[IPv6数据包的条目](https://en.wikipedia.org/wiki/IPv6_packet)。

##Fragmentation (数据分片)

由于底部链路层对所传输的数据帧有最大长度限制（最大传输单元）。所以有时候IPv4需要对所传数据包进行[分片](https://en.wikipedia.org/wiki/IP_fragmentation)。具体表现为，如果数据包尺寸超过了所要经过的数据链路的最大传输限制，路由就会对数据包进行分片。当分片数据包到达目标主机后，可以根据分片信息进行数据重组。当然，数据发送源有权决定路由是否启用对传输数据包进行分片，假如所传输的数据超过了输送限制，又禁止了路由分片，发送源会收到[ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol)(Internet Control Message Protocol，Internet报文控制协议)的*数据帧超长*报告信息。

在IPv6中，如果数据包超限制，路由会直接丢弃数据包并且向发送源回传[ICMP6](https://en.wikipedia.org/wiki/ICMPv6)的*数据帧超长*报告信息。源和目标两端会基于这个特性来进行[路由MTU发现](https://en.wikipedia.org/wiki/Path_MTU)，来寻找两端之间*最大传输单元*（maximum transfer unit）所在的路由。找到MTU路由后，仅当上层数据包的最小payload确实超过了MTU，IPv6才会进行[分片](https://en.wikipedia.org/wiki/IPv6_packet#Fragmentation)传输。对于IPv6下的TCP，这都不是问题。


##TCP- Trasnmission Control Protocol (TCP 传输控制协议)

TCP层位于IP层之上，是最受欢迎的因特网通讯协议之一，人们通常用TCP/IP来泛指整个因特网协议族。

刚刚提到，IP协议允许两个主机之间传送单一数据包（datagrams数据报）。为了保证对所传送数据包达到*尽力服务*的目的，最终的传输的结果可能是数据包乱序、重复甚至丢包。

TCP是基于IP层的协议。但是TCP是可靠的、有序的、有错误检查机制的基于字节流传输的协议。这样当两个设备上的应用通过TCP来传递数据的时候，总能够保证目标接收方收到的数据的顺序和内容与发送方所发出的是一致的。TCP做的这些事看起来稀松平常，但是比起IP层的粗旷处理方式已经是有显著的进步了。

应用程序之间可以通过TCP建立链接。TCP建立的是双向全双工连接，通信双方可以同时进行数据的传输。连接的双方都不需要操心数据是否分块，是否采用了*尽力服务*等。TCP会确保所传输的数据的正确性，即接受方收到的数据与发出方的数据一致。

HTTP是典型的TCP应用。用户浏览器（应用1）与web服务器（应用2）建立连接后，浏览器可以通过连接发送服务请求，web服务器可以通过同样的连接对请求做出响应。

同一个host主机上可以有多个应用同时使用TCP协议。TCP用不同的*端口*来区分应用。作为连接的两端，发送源和接收目标分别拥有自己的IP地址和端口号。凭借这样一对IP地址和端口号，就可以唯一标识一个连接。

使用HTTPS的web服务器会*监听*443端口。浏览器作为发送源会启用一个临时端口结合自己的IP地址与目标服务器对应的端口和IP地址建立TCP连接。

TCP在IPv4和IPv6上是无差别运行的。所以，如果IPv4的*Protocol*(协议区)或IPv6的*Next Hearder*（下一头字段）的协议号被设置成6，表示执行TCP协议。

###TCP Segments (TCP 报文段)

主机之间传输的数据流一般先会被分块，再转化成TCP的报文段，最终会生成IP数据包中的payload载荷数据。

每个TCP报文段都有头信息和对应的载荷payload。payload信息就是待传输的数据块.TCP报文头信息中首要包含的是源和目标端口号，至于说源和目标的IP地址信息是包含在IP头信息中。

TCP的报文头信息中还有报文序列号、确认号等其他一些用于管理连接的信息。

所谓序列号信息，其实就是为每个报文段分配的唯一编号。第一个报文段的序列号是随机的，比如：1721092979，其后的每一个报文段的序列号都以此号为基础依次加1，1721092980，1721092981等等。至于确认号，是目标端反馈给源的确认信息，通知源目前已经接到哪些报文段了。由于TCP是全双工的，所以数据和确认信息发送都是双向的。

### TCP Connections (TCP 连接)

连接管理是TCP的核心功能之一，而且协议需要解决由于IP层采用不可靠传输引发的一系列复杂问题。下面会分别介绍TCP的连接建立、数据传输以及连接终止的详细过程。

TCP连接全过程的状态变化是很复杂的（与T[CP状态图](https://upload.wikimedia.org/wikipedia/commons/f/f6/Tcp_state_diagram_fixed_new.svg)相比）。但是大多数情况下还是比较简单的。

#### Connection Setup (连接建立)

TCP连接都是建立在两个主机之间的。所以，每个连接建立过程中都存在两个角色：一端（例如web服务器）监听连接，另一端（例如应用）主动连接正在监听的一端（web服务器）。服务器端的这种监听行为被称为*passive open*（被动打开）。客户端主动连接服务器的行为被称为*active open*（主动打开）。

TCP会通过三次握手来完成连接建立，具体过程是这样的：

1.客户端首先向服务端发送一个**SYN**包和一个随机序列号A
2.服务端收到后会回复客户端一个**SYN-ACK**包以及一个确认号（用于确认收到SYN）A+1，同时再发送一个随机序列号B
3.客户端收到后会发送一个**ACK**包以及确认号（用于确认收到SYN-ACK）B+1和序列号A+1给服务端

**SYN**是*synchronize sequence numbers*(同步序列号)的缩写。两端在传递数据时，所穿的每个TCP报文段都有一个序列号。就是利用这种机制，TCP可以确保分块传输的数据包最终都以正确的个数和顺序抵达目标端。在正式传输开始之前，源和目标端需要同步确认第一个报文的序列号。

**ACK**是*acknowledgment*（确认）的缩写。当某一端接到了报文包后，通过回传已报文序列号来确认接收到报文这件事。

运行如下语句：
<pre><code class="objectivec">curl -<span class="number">4</span> http:<span class="comment">//www.apple.com/contact/</span></code></pre>

这是通过curl命令与www.apple.com.cn的80端口创建一个TCP连接。

www.apple.com所在服务器23.63.125.15（注意，整个IP不是固定的）会监听80端口。客户端IP地址是10.0.1.6，启用的*临时端口*52181（这个端口是从可用端口中随机选择的）。利用tcpdump(1)输出的三次握手过程是这样的：

<pre><code class="undefined">% sudo tcpdump -c 3 -i en3 -nS host 23.63.125.15
18:31:29.140787 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [S], seq 1721092979, win 65535, options [mss 1460,nop,wscale 4,nop,nop,TS val 743929763 ecr 0,sackOK,eol], length 0
18:31:29.150866 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [S.], seq 673593777, ack 1721092980, win 14480, options [mss 1460,sackOK,TS val 1433256622 ecr 743929763,nop,wscale 1], length 0
18:31:29.150908 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [.], ack 673593778, win 8235, options [nop,nop,TS val 743929773 ecr 1433256622], length 0</code></pre>

这里信息量很大。下面要逐个分析一下。

最左边是系统时间。当时执行命令的时间是晚上18:31。后面的IP代表的是这些都是IP协议数据包。

接下来看这段10.0.1.6.52181 > 23.63.125.15.80，这一对是源和目标端的IP地址＋端口。第一行和第三行是客户端发向服务端的信息，第二行是服务端发向客户端的。tcpdump会自动把端口号加到IP地址后头，比如10.0.1.6.52181表示IP地址为10.0.1.6，端口号为52181。

Flags表示TCP报文段头信息中的一些缩写标识：S代表**SYN**，.代表**ACK**，p代表**PUSH**，F是**FIN**。还有一些其他的标识，这边就不罗列了。注意上面三行Flags中先是携带**SYN**，接着是**SYN-ACK**，最后是**ACK**，这就是三次握手确认的全过程。

另外，第一行中客户端发送了一个随机序列号1721092979 (就是上文所说的A)给服务器。第二行展示的是服务器回传给客户端的确认号1721092980 (A+1) 和一个随机序列号 673593777 (B)。 最后在第三行，客户端将自己的确认号673593778 (B+1)发还给服务端。

####Options (其他选项)

当然，在连接建立过程中还会配置一些其他的信息。比如第一行中客户端发送的内容：

<pre><code class="undefined">[mss 1460,nop,wscale 4,nop,nop,TS val 743929763 ecr 0,sackOK,eol]</code></pre>

还有第二行服务端发送的：

<pre><code class="undefined">[mss 1460,sackOK,TS val 1433256622 ecr 743929763,nop,wscale 1]</code></pre>

其中TS val / ecr是TCP用来创建round-trip time（RTT往返时间）的。TS val是发送方的*time stamp*（时间戳），ecr是*echo reply*（相应应答）时间戳，通常情况下就是发送方收到的最后时间戳。TCP以RTT作为其拥塞控制算法（congestion-control algorithms）的依据。

连接的两端都发送sackOK。这样会启用*Selective Acknowledgement*(选择性确认)机制，使连接双方能够确认收到的字节范围。一般情况下，确认机制只是确认接受方已收到的数据的字节总数。[RFC 2018第3部分](http://tools.ietf.org/html/rfc2018#section-3)有对SACK的详细阐述。

mss选项声明了*Maximum Segment Size*(最大报文长度)，表示接收端希望接收的单个报文的最大长度（以字节为单位）。wscale是*window scale factor*(窗口放大因子)，稍后会详细说明。

#### Connection Data Flow (数据传输)

一旦建立了连接，双方就可以互发数据了。发送端所发出的每个报文段都有一个序列号，这个序列号与当下已传送的字节总数有关。接收端会针对已接收的数据包向源端发送确认报文，确认信息同样是报文头携带**ACK**。

假设现在传送的信息是除最后一个报文5字节外，其他都是10字节。具体是这样的：

<pre><code class="undefined">host A sends segment with seq 10
host A sends segment with seq 20
host A sends segment with seq 30    host B sends segment with ack 10
host A sends segment with seq 35    host B sends segment with ack 20
                                    host B sends segment with ack 30
                                    host B sends segment with ack 35</code></pre>
                                    
整个机制是双向运转的。A主机会持续的发送数据包。B收到数据包后会向A发送确认信息。A发送数据包的过程不需要等待B的确认。

TCP将流量控制和其他一系列复杂机制结合起来进行拥塞控制。需要处理以下问题：针对丢失的报文采用重发机制，同时还需要动态的调整发送报文的频率。

流量控制的原则是发送方发送数据的速度不能比接收方处理数据的速度快。接收方也就是所谓的*receive window*（接收窗口）会告知发送方自身接收窗口数据缓冲区的大小。从上面tcpdump的输出来看，win（窗口大小）是65535，wscale（窗口放大因子）是4。这些数字的意思是说，10.0.1.6主机的接收窗口大小是4＊64 kB = 256 kB，23.63.125.15主机的win是14480，wscale是1，接收窗口约为14KB。总之，不管哪一方作为数据接收方，都会向对方通报自己的接收窗口大小。

拥塞控制要更复杂一些。所有拥塞控制的目标都是要计算出当前网络中数据传输的最佳速率。所谓最佳速率就是要达到一种微妙的平衡。一方面，是希望速度越快越好，另一方面，速度快意味着数据传输多，这样处理性能会大打折扣甚至导致崩溃。而这种[超负荷崩溃](https://en.wikipedia.org/wiki/Congestive_collapse#Congestive_collapse)是分组交换网络的固有特点。当负载过大，数据包之间会产生拥塞，直接导致丢包率急速上升。

拥塞控制还需要充分考虑对流量的影响。[RFC 5681](https://www.rfc-editor.org/rfc/rfc5681.txt)中对TCP拥塞控制有6,000字左右的阐述。发送方要时刻关注来自接收方的确认信息。要做到这点并不简单，有的时候还需要一定的妥协。要知道底部IP协议数据包是无序传输的，数据包会丢失也会重复。发送方需要评估RTT往返时间，然后基于RTT去确定是否收到了接收方的确认信息。重发数据包也有很大代价，除了连接延迟问题，网络的负载也会发生明显的波动。导致TCP需要不停的去适应当前网络情况。

更重要的是，TCP连接本身是易变的。除了数据传输，连接的两端还会不时的发送一些提醒和确认信息以便可以适当的调整状态来维持连接。

基于这种一直在相互协调中的连接关系，TCP连接往往会是短暂而低效的。在建立连接的初期，TCP协议算法还不能完全了解当前网络状况。而在连接将要结束的时候，反馈给发送方的信息又可能不充分，这样就很难对连接状况做出实时的合理的评估。

之前展示了客户端和服务端之间交换的三段报文。再看看关于连接的其他信息：

<pre><code class="undefined">18:31:29.150955 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [P.], seq 1721092980:1721093065, ack 673593778, win 8235, options [nop,nop,TS val 743929773 ecr 1433256622], length 85
18:31:29.161213 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], ack 1721093065, win 7240, options [nop,nop,TS val 1433256633 ecr 743929773], length 0</code></pre>

客户端10.0.1.6发送的第一段报文长度是85 bytes（HTTP请求）。由于在上一个报文发送后没有收到来自服务端的信息，所以ACK确认号的值不变。

服务端23.63.125.15只是对接收客户端的数据进行确认回复，没有向客户端发送数据，所以length为0。由于当前连接是采用*Selective acknowledgments*(选择性确认)，所以序列号和确认号是之间的字节长度是从1721092980到1721093065，也就是85 bytes。接收方发送的ACK确认号是1721093065，这代表目前已接收的数据确认累计到1721093065字节了。至于说为什么数字会如此之大，这和初次握手发出的随机数有关，当前数字减掉初始随机数才是累计收到的数据包真正的大小。

这种模式会一直持续到全部数据传送完成：

<pre><code class="undefined">18:31:29.189335 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673593778:673595226, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.190280 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673595226:673596674, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.190350 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [.], ack 673596674, win 8101, options [nop,nop,TS val 743929811 ecr 1433256660], length 0
18:31:29.190597 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673596674:673598122, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.190601 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673598122:673599570, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.190614 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673599570:673601018, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.190616 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673601018:673602466, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.190617 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673602466:673603914, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.190619 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673603914:673605362, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.190621 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673605362:673606810, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.190679 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [.], ack 673599570, win 8011, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
18:31:29.190683 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [.], ack 673602466, win 7830, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
18:31:29.190688 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [.], ack 673605362, win 7830, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
18:31:29.190703 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [.], ack 673605362, win 8192, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
18:31:29.190743 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [.], ack 673606810, win 8192, options [nop,nop,TS val 743929812 ecr 1433256660], length 0
18:31:29.190870 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [.], seq 673606810:673608258, ack 1721093065, win 7240, options [nop,nop,TS val 1433256660 ecr 743929773], length 1448
18:31:29.198582 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [P.], seq 673608258:673608401, ack 1721093065, win 7240, options [nop,nop,TS val 1433256670 ecr 743929811], length 143
18:31:29.198672 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [.], ack 673608401, win 8183, options [nop,nop,TS val 743929819 ecr 1433256660], length 0</code></pre>


#### Connection Termination (终止连接)

最终连接会终止（或结束）。连接的每一端都会发送**FIN**标识给另一端来声明结束传输，接着另一端会对收到**FIN**进行确认。当连接两端均发送完各自**FIN**和做出相应的确认后，连接将会彻底关闭：

<pre><code class="undefined">18:31:29.199029 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [F.], seq 1721093065, ack 673608401, win 8192, options [nop,nop,TS val 743929819 ecr 1433256660], length 0
18:31:29.208416 IP 23.63.125.15.80 &gt; 10.0.1.6.52181: Flags [F.], seq 673608401, ack 1721093066, win 7240, options [nop,nop,TS val 1433256680 ecr 743929819], length 0
18:31:29.208493 IP 10.0.1.6.52181 &gt; 23.63.125.15.80: Flags [.], ack 673608402, win 8192, options [nop,nop,TS val 743929828 ecr 1433256680], length 0</code></pre>

这里值得注意的是第二行，23.63.125.15发送了**FIN**，同时在这个报文信息中还对第一行中另一端发送的**FIN**予以**ACK**（以.代表）确认。


##HTTP — Hypertext Transfer Protocol (超文本传输协议)

1989年，蒂姆.伯纳斯.李(Tim Berners Lee)在[CERN](https://en.wikipedia.org/wiki/CERN)(European Organization for Nuclear Research欧洲原子核研究委员会)担任软件咨询师的时候，开发了一套程序，奠定了[World Wide Web](https://en.wikipedia.org/wiki/World_Wide_Web)(万维网)的基础。*HyperText Transfer Protocol*（超文本转移协议，即HTTP）是用于从WWW服务器传输超文本到本地浏览器的传送协议。[RFC 2616](https://tools.ietf.org/html/rfc2616)定义了今天普遍使用的一个版本：HTTP 1.1。

###Request and Response (请求与响应)

HTTP采用简单的请求和响应机制。在safari输入http://www.apple.com时，会向www.appple.com所在的服务器发送一个HTTP请求。服务器会对请求做出一个响应，将请求结果信息返回给safari。

每一个请求都有一个对应的响应信息。请求和响应遵从同样的格式。第一行是*request line*（请求行）或者*status line*（响应状态行）。接下来是header头信息，header信息之后会有一个空行。空行之后是body请求信息体。

###一个简单请求

当[Safari](https://en.wikipedia.org/wiki/Safari_%28web_browser%29)加载HTML页面http://www.objc.io/about.html的时候，先是发送HTTP请求到www.objc.io，请求的内容是：

<pre><code class="objectivec">GET /about<span class="variable">.html</span> HTTP/<span class="number">1.1</span>
Host: www<span class="variable">.objc</span><span class="variable">.io</span>
Accept-Encoding: gzip, deflate
Connection: keep-alive
If-None-Match: <span class="string">"a54907f38b306fe3ae4f32c003ddd507"</span>
Accept: text/html,application/xhtml+xml,application/xml;q=<span class="number">0.9</span>,*<span class="comment">/*;q=0.8
If-Modified-Since: Mon, 10 Feb 2014 18:08:48 GMT
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_2) AppleWebKit/537.74.9 (KHTML, like Gecko) Version/7.0.2 Safari/537.74.9
Referer: http://www.objc.io/
DNT: 1
Accept-Language: en-us</span></code></pre>

第一行是**请求行**。它包含三部分信息：动作，资源信息，还有HTTP的版本。

本例中，动作是GET。所谓动作也就是常说的HTTP[请求方法](https://en.wikipedia.org/wiki/HTTP_method#Request_methods)。资源信息表明所请求的资源。例子中的资源信息是/about.html，表示向服务器*get*请求的信息在/about.html中。当前HTTP版本是HTTP/1.1。

接下来10行是HTTP header信息。跟着是一行空行。例子中的请求没有body信息。

header的作用是向服务器传递一些额外的辅助信息，它的内容比较宽泛。维基百科中有[常用HTTP header关键字](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields)信息的清单。例子中的header信息host：www.objc.io表示告诉服务器，本次请求的服务器名称是什么。这样可以让同一个服务器处理针对多个[域名](https://en.wikipedia.org/wiki/Domain_names)的请求。

下面是一些常见的header信息:

<pre><code class="objectivec">Accept: text/html,application/xhtml+xml,application/xml;q=<span class="number">0.9</span>,*<span class="comment">/*;q=0.8
Accept-Language: en-us</span></code></pre>

服务器可能具备返回多种媒体类型的能力，Accept表示Safari希望接收的媒体格式类型。text/html是[互联网媒体类型](https://en.wikipedia.org/wiki/Mime_type)(Internet media types)，也被称为MIME(MIME是法语)类型或者是内容类型（Content-types）。q=0.9表示Safari对给定媒体类型的优先级要求。Accept-Language代表Safari希望接收的自然语言清单。这里是要求服务器尽可能的根据清单要求去匹配相应的语言。

<pre><code class="undefined">Accept-Encoding: gzip, deflate</code></pre>

Accept-Encoding表示服务器可以对响应body做压缩处理。如果header信息中没有设置压缩标识，那么服务器就不可以返回压缩后的信息。压缩可以大大减少数据的传输量，在文本信息（比如HTML）中尤为明显。

<pre><code class="undefined">If-Modified-Since: Mon, 10 Feb 2014 18:08:48 GMT
If-None-Match: "a54907f38b306fe3ae4f32c003ddd507"</code></pre>

这两行信息表明Safari已经对请求结果做过缓存。如果服务器上的待请求内容在2月10号以后发生过变化或者是ETag与a54907f38b306fe3ae4f32c003ddd507不匹配，这就表示请求结果与当前缓存信息不一致，需要服务器返回最新的请求结果。

User-Agent是告知服务器当前发送请求的客户端类型。

###一个简单响应

作为上面请求的响应，服务器的返回是：

<pre><code class="objectivec">HTTP/<span class="number">1.1</span> <span class="number">304</span> Not Modified
Connection: keep-alive
Date: Mon, <span class="number">03</span> Mar <span class="number">2014</span> <span class="number">21</span>:<span class="number">09</span>:<span class="number">45</span> GMT
Cache-Control: max-age=<span class="number">3600</span>
ETag: <span class="string">"a54907f38b306fe3ae4f32c003ddd507"</span>
Last-Modified: Mon, <span class="number">10</span> Feb <span class="number">2014</span> <span class="number">18</span>:<span class="number">08</span>:<span class="number">48</span> GMT
Age: <span class="number">6</span>
X-Cache: Hit from cloudfront
Via: <span class="number">1.1</span> eb67cb25620df959ba21a943fbc49ef6<span class="variable">.cloudfront</span><span class="variable">.net</span> (CloudFront)
X-Amz-Cf-Id: dDSBgR86EKBemW6el-pBI9kAnuYJEaPQYEqGmBnilD12CbixCuZYVQ==</code></pre>

第一行是*status line*(状态行)。它包括HTTP版本，[状态码](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)（304）和状态信息。

HTTP定义了[一系列状态码](https://en.wikipedia.org/wiki/Http_status_codes)，它们各有用途。本例中的304表示所请求的信息自上次访问以来没有变化。响应中不应包含body信息。也就是说服务器通知客户端：建议直接使用当前缓存信息。

###关闭缓存

用curl发送一个请求：

<pre><code class="objectivec">% curl http:<span class="comment">//www.apple.com/hotnews/ &gt; /dev/null</span></code></pre>

curl没有使用本地缓存。整个请求会是这样的：

<pre><code class="objectivec">GET /hotnews/ HTTP/<span class="number">1.1</span>
User-Agent: curl/<span class="number">7.30</span><span class="number">.0</span>
Host: www<span class="variable">.apple</span><span class="variable">.com</span>
Accept: *<span class="comment">/*</span></code></pre>

这个请求与之前Safari发的请求很类似。但是curl请求的header信息中没有If-None-Match，所以服务器必须将请求结果返回。

此处curl头信息中声明的Accept: */* 表示可以接收任何媒体类型。

来自www.apple.com的响应：

<pre><code class="undefined">HTTP/1.1 200 OK
Server: Apache
Content-Type: text/html; charset=UTF-8
Cache-Control: max-age=424
Expires: Mon, 03 Mar 2014 21:57:55 GMT
Date: Mon, 03 Mar 2014 21:50:51 GMT
Content-Length: 12342
Connection: keep-alive

&lt;!DOCTYPE html&gt;
&lt;html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en-US" lang="en-US"&gt;
&lt;head&gt;
    &lt;meta charset="utf-8" /&gt;</code></pre>
    
收到的响应信息body中包含了HTML信息。

Apple服务器响应的[状态码](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)是200，这表示HTTP请求成功。

服务器同时还告知响应媒体类型是 text/html；字符集 charset=UTF-8；内容长度Content-Length：12342，代表body信息的大小。

##HTTPS - HTTP Secure (HTTP安全)

[Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security)（安全传输层协议）是一种基于TCP的加密协议。它支持两件事：传输的两端可以互相验证对方的身份，加密传输数据。基于TLS的HTTP请求就是HTTPS。

用HTTPS去替代HTTP，在安全方面会有显著的提升。也许你还会采用一些其他的安全措施，总之这都会为安全通信提供保障。

###TLS 1.2

如果服务器采用[TLS 1.2](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_1.2)版本，需要用kTLSProtocol12去替换TLSMinimumSupportedProtocol。这能有效的防御[man-in-the-middle attacks](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)（中间人攻击）。

### Certificate Pinning (证书锁定)

如果不确定数据接收方的身份，那么即便对所传输数据进行加密也没什么意义。由于服务器的证书可以表明服务器的身份，只允许和持有某个特定证书的一方建立连接就是[certificate pinning](https://en.wikipedia.org/wiki/Certificate_pinning#Certificate_pinning)（证书锁定）。

如果一个客户端通过TLS和服务器建立连接，操作系统会验证服务器证书的有效性。当然，有很多手段可以绕开这个校验，最直接的是在iOS设备上安装证书并且将其设置为可信的。这种情况下，实施[中间人攻击](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)也不是什么难事。

可以使用证书锁定来规避这种风险（或者说是将风险降到最低）。当建立TLS连接后，应立即检查服务器的证书，不仅要验证证书的有效性，还需要确定证书和其持有者是否匹配。考虑到应用和服务器需要同时升级证书的要求，这种方式比较适合应用在访问自家服务器的情况下。

为了实现证书锁定，在建立连接的过程中需要对服务器进行检查(*server trust*)。每当通过 NSURLSession创建了连接，NSURLSession的代理就会收到一个调用 -URLSession:didReceiveChallenge:completionHandler:。传递的参数 NSURLAuthenticationChallenge有一个属性 protectionSpace，protectionSpace是 NSURLProtectionSpace的实例，它有serverTrust属性。

serverTrust是SecTrustRef对象。安全框架提供了很多方法用于验证SecTrustRef。AFNetworking中的[AFSecurityPolicy](https://github.com/AFNetworking/AFNetworking/blob/7f2c395ba185b586468557b22977ccd2b79fae66/AFNetworking/AFSecurityPolicy.m)就是一个不错的应用。一如既往的提醒大家，如果要自己构建安全验证相关的代码，请一定要认真做好代码走查，千万不要再出现诸如[goto_fail](https://www.imperialviolet.org/2014/02/22/applebug.html)这类bug。

##综合讨论

现在大家对IP,TCP和HTTP的工作原理有了一定的了解了。下面说说还可以做些什么以及一些相关注意事项。

###有效的使用连接

TCP连接容易在两个时点出现问题：初始设置，以及通过连接传输的最后一部分报文。

####设置

连接设置可能会非常耗时。正如前文所说，TCP建立连接的过程中需要进行三次握手。这个过程中本身没有太多的数据需要传递。但是，对于移动网络来说，从手机端向服务器端发送一个数据包普遍需要250ms，也就是四分之一秒。推及到三次握手，也就是说在还没有传送任何数据之前，光建立连接就要花费750ms。

HTTPS的情况更夸张，由于HTTPS是基于TLS的HTTP，而HTTP又基于TCP。只要是TCP连接就要执行三次握手。然后到了TLS层还会再握手三次。估算一下，建立一个HTTPS连接的耗时至少是创建一个HTTP连接的两倍。如果RTT时间是500 ms（假设单程250 ms），HTTPS建立连接累计总耗时将达1.5秒。

不管建立连接后是要传递多少数据，建立连接本身都太过耗时了。

另一个影响TCP连接的因素是传送大规模数据。如果要在网络情况未知的条件下传送报文，TCP需要侦测当前网络的能力。换句话说，TCP得花费一定的时间去计算此网络的最佳传输速率。上文提到过，TCP需要逐步调整以便找到最佳速度。这种算法被称为[slow-start](https://en.wikipedia.org/wiki/Slow-start)（慢启动）。还有一点值得注意，慢启动策略在那些数据链路层传输质量较差的网络环境中的表现更差，而无线网络就是典型的例子。

####丢包问题

数据传输也存在一些问题。每当客户端发起HTTP请求某些资源的时候，服务器会持续的向客户端主机发送TCP报文数据，客户端收到报文后需要给服务器反馈ACK确认信息。假如某个报文在传输过程中发生丢包，那么服务器也就不会收到该报文的确认ACK。一旦服务器发现有报文没有ACK反馈，就会触发[快速重传](https://en.wikipedia.org/wiki/Fast_retransmit)（fast retransmit）。

每当某个数据包丢失，数据接收方在收到下个数据包后发出的确认ACK与所接收的前一个数据包确认ACK相同。那么数据发送方自然就会收到重复的ACK。除了报文丢失，还有很多种网络状况会导致重复ACK的问题。一般情况下，如果数据发送方连续收到3个重复的ACK就会立即进行快速重发。

在数据传输的收尾阶段，如果发送方完成数据发送，接受方自然会停止发送ACK确认。但是在最后四个报文传输的过程中，快速重发算法没有办法处理这四个报文的数据包丢失问题（不会收到三个相同的确认ACK，不能界定传输丢包）。在常规网络环境下，四个数据包相当于5.7KB的数据规模。总之，在最后四个报文传输的过程中，快速重发机制是无效的。针对这种情况，TCP会启用其他机制来侦测丢包问题，接着可能要消耗几秒钟去执行报文重发。


###长连接及并行处理

HTTP有两种策略来解决这些问题。最简单的是[HTTP持久连接](https://en.wikipedia.org/wiki/HTTP_persistent_connection)(persistent connection)，也被称为*长连接*(keep-alive)。具体就是，每当HTTP完成一组请求－响应处理后，还会继续复用相同的TCP连接。而HTTPS会复用同样的TLS连接：

<pre><code class="undefined">open connection
client sends HTTP request 1 -&gt;
                            &lt;- server sends HTTP response 1
client sends HTTP request 2 -&gt;
                            &lt;- server sends HTTP response 2
client sends HTTP request 3 -&gt;
                            &lt;- server sends HTTP response 3
close connection</code></pre>

第二步就利用了[HTTP pipelining](https://en.wikipedia.org/wiki/Http_pipelining)（并行）处理，即允许客户端利用同样的连接并行发送多个请求，也就是说无需等待上一个请求的响应完成可以发下一个请求。这表示能同时处理请求和响应，请求处理的顺序采用[先进先出](https://en.wikipedia.org/wiki/FIFO)原则，响应结果会按照请求发出的顺序依次返还给客户端。

进一步简化展示：

<pre><code class="undefined">open connection
client sends HTTP request 1 -&gt;
client sends HTTP request 2 -&gt;
client sends HTTP request 3 -&gt;
client sends HTTP request 4 -&gt;
                            &lt;- server sends HTTP response 1
                            &lt;- server sends HTTP response 2
                            &lt;- server sends HTTP response 3
                            &lt;- server sends HTTP response 4
close connection</code></pre>

注意，服务器发出的响应是实时的，不会等到接收完全部请求才处理。

可以利用这个特点来提升TCP的效率。只需要在建立连接初始阶段执行一次握手（握三下），而后一直复用同样的连接，这样TCP就可以最大限度的利用带宽。此种情况下，拥塞控制也会随之提升。因为快速重发机制无法处理的最末四个报文丢失情况只会发生在使用本连接的最后一个请求－响应中，而不是像之前那样每一个请求－响应都需要建立自己的连接，每个连接中都可能出现最后四个报文丢失的问题。

HTTP pipelining对高网络延迟连接的通讯性能提升尤为显著，如果iPhone不是通过WI-FI访问网络，那么此类网络连接就属于高延迟范畴。实际上，有[调查](http://research.microsoft.com/pubs/170059/A%20comparison%20of%20SPDY%20and%20HTTP%20performance.pdf)显示，在移动网络环境下，[SPDY](https://en.wikipedia.org/wiki/SPDY)的通讯性能并不优于HTTP pipelining。

[RFC 2616](https://tools.ietf.org/html/rfc2616)指明，在与同一个服务器通讯的时候，如果启用了HTTP pipelining，建议启用两个连接。按照说明所述，这样能获得最优响应效率，能最大限度避免拥塞。而且，增加更多的连接也不会再对性能有什么明显改善。

遗憾的是，还是有相当多的服务器不支持pipelining。由于这个原因，HTTP pipelining在NSURLSession中默认是关闭的。如果想要启用HTTP pipelining，需要将NSURLSessionConfiguration中的HTTPShouldUsePipelining设置为YES。另外，建议服务器最好还是支持pipelining。

###超时处理

我们都有在网络不太好的情况下使用app的经历。很多app大概15秒左右就会结束请求并且反馈一个超时信息。这种设计其实是很不友好的。应该给用户一个他们可以理解的友好提示，诸如“你好，现在网络状况不太好，您需要多等一会儿。”。但是即便网络状况不好，只要连接还在，TCP都会保证将请求发出去并且会一直等待响应的返回，只是时间长短的问题。

从另一个角度来说：在较慢的网络中，请求－响应的RTT时间可能会有17秒。如果15秒就决定中止请求，就算用户有足够的耐心，他们也没机会等到想要的操作结果。反过来，如果我们给出用户相应的提示信息，而他们又刚好愿意多等一会，用户可能会更喜欢使用这样的应用。

一直以来都有一种误解，用重发请求来解决上面的问题。注意，这不是问题的关键，因为TCP有自己的重发机制。

正确的处理方式应该是：每当发起一个请求的时候，同时启动一个10秒计时器。如果请求在10秒之内返回，就把计时器停掉。如果超过10秒，可以给用户一个提示“网络不好，请稍后。”，我建议再给用户一个取消按钮，让他们可以自行选择等待还是取消请求，当然提示信息的具体内容和是否配备取消按钮，这个可以视乎各app的情况去决定。总而言之，开发者最好不要直接替用户做决定，比如直接中止他们的请求。

只要连接双方的IP地址是不变的、可用的，连接就一定会是“活跃”的。如果把iPhone从WI-FI连接切换到3G网络，这样连接就会变得不可用，因为手机的IP地址发生了变化，基于原IP地址创建的路由自然是失效的。

###缓存

看看第一个例子中发送的这段header信息：

<pre><code class="undefined">If-None-Match: "a54907f38b306fe3ae4f32c003ddd507"</code></pre>

这表示客户端本地已经针对所请求的资源做过缓存了，如果服务器上的资源有过更新，需要将最新的资源返回给客户端，否则不需要返回。如果自己构建客户端和服务器的数据通信，建议充分利用这个机制。它的名称是HTTP [ETag](https://en.wikipedia.org/wiki/HTTP_ETag)，如果使用得当，会对通讯的效率有明显的优化。

记住“最快的请求是不发请求”。举个极端的例子，拿一个请求来说，哪怕你有最好的网络，请求的数据量极小，有超快的服务器，你也不大可能在50ms内拿到请求的响应。这还只是一个请求。想想吧，如果在本地创建相同的数据耗时小于50ms，你完全没必要发这个请求。

针对已请求的资源，只要服务器上对应的资源具备在一定时间内不发生变化特性，建议在本地缓存起来。注意检查header中缓存过期的相关属性，也可以直接利用NSURLSession中的NSURLRequestUseProtocolCachePolicy策略。

##总结

利用NSURLSession发HTTP请求是非常简单便捷的。但是请求背后有很多技术点做支撑。只有知晓和理解其中的细节和内涵才能更好的去优化HTTP请求。用户期望的是我们的app时时刻刻都是好用的。只有深刻理解IP,TCP和HTTP的工作原理才能更好的去满足用户的期望。















































