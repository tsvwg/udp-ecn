---
title: "Configuring UDP Sockets for ECN for Common Platforms"
abbrev: "udp-ecn"
category: info
ipr: trust200902

docname: draft-ietf-tsvwg-udp-ecn-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
date: {DATE}
area: "Web and Internet Transport"
workgroup: "Transport and Services Working Group"
keyword:
 - udp
 - ecn
venue:
  group: "Transport and Services Working Group"
  type: "Working Group"
  mail: "tsvwg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tsvwg/"
  github: "tsvwg/udp-ecn"
  latest: "https://tsvwg.github.io/udp-ecn/draft-ietf-tsvwg-udp-ecn.html"

author:
 -
    fullname: Martin Duke
    organization: Google
    email: martin.h.duke@gmail.com

informative:
 APPLE-NETWORK-FRAMEWORK:
  title: "NWProtocolIP.Metadata"
  target: "https://developer.apple.com/documentation/network/nwprotocolip/metadata"

 CHROMIUM:
   title: "The Chromium Projects"
   target: "https://www.chromium.org/chromium-projects/"

 CHROMIUM-POSIX:
   title: "udp_socket_posix.cc"
   target: "https://source.chromium.org/chromium/chromium/src/+/main:net/socket/udp_socket_posix.cc"

 CHROMIUM-WINDOWS:
   title: "udp_socket_win.cc"
   target: "https://source.chromium.org/chromium/chromium/src/+/main:net/socket/udp_socket_win.cc"

 WINDOWS-DOC:
   title: "WSASetRecvIPEcn function (ws2tcpip.h)"
   target: "https://learn.microsoft.com/en-us/windows/win32/api/ws2tcpip/nf-ws2tcpip-wsasetrecvipecn"

 WINDOWS-SOCKOPT:
   title: "MSDN - IPPROTO_IP socket options"
   target: "https://learn.microsoft.com/en-us/windows/win32/winsock/ipproto-ip-socket-options"

--- abstract

Explicit Congestion Notification (ECN) applies to all transport protocols in
principle. However, it had limited deployment for UDP until QUIC became widely
adopted. As a result, documentation of UDP socket APIs for ECN on various
platforms is sparse. This document records the results of experimenting with
these APIs in order to get ECN working on UDP for Chromium on Apple, Linux, and
Windows platforms.

--- middle

# Introduction

{{?RFC3168}} defines a two-bit field in the IP header for Explicit Congestion
Notification (ECN), which provides network feedback to endpoint congestion
controllers. This has historically mostly been relevant to TCP ({{?RFC9293}}),
where any incoming ECN field are internally consumed by the kernel, and
therefore imply no application interface except enabling and disabling the
capability.

The Stream Control Transport Protocol (SCTP) ({{?RFC9260}}) has long supported
ECN in its design. SCTP is sometimes carried over DTLS and UDP ({{?RFC8261}}).
In principle, user-space implementers might have leveraged UDP ECN APIs to
deliver ECN field between SCTP and the UDP socket. At the time of
publication, the TSV Working Group is not aware of any such efforts.

{{?RFC6679}} defines ECN over RTP over UDP. The Working Group is aware of a
research implementation, but cannot confirm any commercial deployments.

However, QUIC {{?RFC9000}} runs over UDP and has seen wider deployment than
SCTP. The Low Latency, Low Loss, Scalable Throughput (L4S) experiment
({{?RFC9330}}) and QUIC have combined to increase interest in ECN over UDP.

The Chromium Projects ({{CHROMIUM}}) provide a widely-deployed protocol library
that includes QUIC. An effort to provide ECN support for QUIC on the many
platforms on which Chromium is deployed revealed that many ECN-related UDP
socket interfaces are poorly documented.

This informational document provides a record of that experience, to encourage
further support for ECN in other QUIC implementations, and indeed any consumer
of ECN field that operates over UDP. It is not a standards-track document
and does not bind platforms to any API, or suggest any such API.

Many socket APIs continue to reference the "ToS (Type of Service) byte",
including the IP_TOS label, even though {{?RFC2474}} obsoleted that in 1998.
That 8-bit field now contains a 6-bit Differentiated Services Code Point (DSCP)
and the 2-bit ECN field.

This document focuses on the APIs for the C and C++ languages. Other languages
are likely to have different syntax and capabilities.

# Conventions and Definitions

This document is not a general tutorial on UDP socket programming, and assumes
familiarity with basic socket concepts like binding, socket options, and
common system error codes.

Throughout this document, "Apple" refers to both macOS and iOS.

# Receiving the ECN field

Network devices can change the ECN field in the IP header. Since this
feedback is required at the packet sender, the packet receiver needs to extract
this codepoint from the UDP socket in order to report to the sender.

There are two components to this: setting the socket to report incoming ECN
field, and retrieving the ECN field for each incoming packet.

Note that Apple platforms additionally provide a framework for network
connections that allows receiving the ECN field when using UDP without traditional
socket option semantics. When sending or receiving UDP datagrams, IP protocol
metadata carries ECN information in both directions. See
{{APPLE-NETWORK-FRAMEWORK}}.

## Setting the socket to report incoming ECN fields

### Linux, Apple, and FreeBSD

To receive the ECN field, applications set a socket option to true using a setsockopt()
call.

On all platforms, IPv4 sockets require the IPPROTO_IP-level socket option with
name IP_RECVTOS to be set.

On all platforms, IPv6 sockets require the IPPROTO_IPV6-level socket option with
name IPV6_RECVTCLASS to be set.
If the IPv6 socket is not IPv6 only, on Linux hosts it is required to also set
the IPPROTO_IP-level socket option IP_RECVTOS to receive the ECN field for
UDP/IPv4 packets.

At the time of writing, an example implementation can be found at
{{CHROMIUM-POSIX}}.

### Windows

Windows documentation recommends using the function WSASetRecvIPEcn() to
enable ECN field reporting regardless of the IP version. This function dates to
Windows 10 Build 20348, according to {{WINDOWS-DOC}}.

However, this can also be accomplished by calling setsockopt() and using
options of level IPPROTO_IP and name IP_RECVECN for IPv4, and IPPROTO_IPV6
and IPV6_RECVECN for IPv6. These options are documented at
{{WINDOWS-SOCKOPT}}.

For IPv6 sockets which are not IPv6 only, WSASetRecvIPEcn() will not enable ECN reporting for
IPv4. This requires a separate setsockopt() call using the IP_RECVECN option.

If a socket is bound to a IPv4-mapped IPv6 address (i.e. it is of the format
::ffff:&lt;IPv4 address&gt;), calls to WSASetRecvIpEcn() return error EINVAL.
These sockets should instead use an explicit setsockopt() call to set
IP_RECVECN.

At the time of writing, an example implementation can be found at
{{CHROMIUM-WINDOWS}}.

## Retrieving ECN fields on incoming packets

All platforms described in this document require the use of a recvmsg() call to
read data from the socket to retrieve the ECN field, because that information
is provided as ancillary data.
Those platforms all return zero or more "cmsg"s that contain requested information
about the arriving packet.

Examples of the technique described below can be found at {{CHROMIUM-POSIX}}
and {{CHROMIUM-WINDOWS}}.

### Linux

If a UDP/IPv4 message is received, Linux will include a cmsg of level IPPROTO_IP
and type IP_TOS. The cmsg data contains an unsigned char.
This applies to IPv4 sockets and IPv6 socket, which are not IPv6 only.

If a UDP/IPv6 message is received, Linux will include a cmsg of level IPPROTO_IPV6
and type IPV6_TCLASS. The cmsg data contains an int.
This applies to IPv6 sockets.

The cmsg data contains the entire IP header byte, which includes the DSCP
and ECN field.
The ECN field constitutes the two least-significant bits of this byte.

The same applies to the Linux-specific recvmmsg() call.

### Apple and FreeBSD

If a UDP/IPv4 message is received on an IPv4 socket, the ancillary data
will contain a cmsg of level IPPROTO_IP and type IP_RECVTOS. The cmsg data
contains an unsigned char.

If a UDP/IPv6 or UDP/IPv4 message is received on an IPv6 socket, the
ancillary data will contain a cmsg of level IPPROTO_IPV6 and type IPV6_TCLASS.
The cmsg data contains an int.

The cmsg data contains the entire IP header byte, which includes the DSCP
and ECN field.
The ECN field constitutes the two least-significant bits of this byte.

### Windows

If the incoming packet is UDP/IPv4, the socket will include a cmsg of level
IPPROTO_IP and type IP_ECN. The cmsg data contains an int.

If the incoming packet is UDP/IPv6, the socket will include a cmsg of level
IPPROTO_IPV6 and type IPV6_ECN. The cmsg data contains an int.

The cmsg data solely consists of the ECN field, and requires no
further bitwise operations.

# Sending ECN codepoints

Existing ECN specifications ({{RFC3168}}, {{RFC9330}}} envision a particular
connection consistently sending the same ECN field. It might transition that
marking after successfully completing a handshake, recognizing the path or the
peer do not support ECN, or transitioning to a new path. Therefore, using a
socket option to configure a consistent marking is generally more resource-
efficient.

However, some server designs receive all incoming packets on a single socket.
As the many connections that constitute this packet stream may have different
support for ECN, it is suitable to provide the ECN field on a per-packet basis.

Note that Apple platforms additionally provide a framework for network
connections that allows sending the ECN field when using UDP without traditional
socket option semantics. When sending or receiving UDP datagrams, IP protocol
metadata carries ECN information in both directions. See
{{APPLE-NETWORK-FRAMEWORK}}.

## On a per-socket basis

### Apple, FreeBSD, and Linux

For sending UDP/IPv4 packets on an IPv4 socket, Apple, FreeBSD, and Linux platforms
allow the outgoing ECN field to be configured by using the IPPROTO_IP-level socket
option with name IP_TOS.
The value has the type int.

For sending UDP/IPv6 packets on an IPv6 socket, Apple, FreeBSD, and Linux platforms
allow the outgoing ECN field to be configured by using the IPPROTO_IPV6-level socket
option with name IPV6_TCLASS.
The value has the type int.

For sending UDP/IPv4 packets on an IPv6 socket, Linux platforms allow the
the outgoing ECN field to be configured by using the IPPROTO_IP-level socket
option with name IP_TOS.

For sending UDP/IPv4 packets on an IPv6 socket, Apple and FreeBSD platforms allow
the outgoing ECN field to be configured by using the IPPROTO_IPV6-level socket
option with name IPV6_TCLASS. On Apple platforms, only the ECN field is taken
into account.

In almost all cases, this setsockopt() call also sets the Differentiated Services
Code Point (DSCP) that make up the rest of the header byte.
Applications making this call will generally want to preserve any existing DSCP
setting, which might require an additional getsockopt() call.

An example of the technique described above can be found at {{CHROMIUM-POSIX}}.

### Windows

At the time of this writing, Windows does not provide a way to configure
marking on a per-socket basis.

## On a per-packet basis

Packets can be individually marked with an ECN field using the ancillary data
that accompanies a sendmsg() call.

### Apple, FreeBSD, and Linux

For sending UDP/IPv4 packets on an IPv4 socket, Apple, FreeBSD, and Linux use
a cmsg with level IPPROTO_IP and type IP_TOS. On Apple and Linux the type of
data is int and for FreeBSD it is unsigned char.

For sending UDP/IPv6 packets on an IPv6 socket, Apple, FreeBSD, and Linux use
a cmsg with level IPPROTO_IPV6 and type IPV6_TCLASS. The type of the data
is int.

For sending UDP/IPv4 packets on an IPv6 socket, Linux requires a cmsg with
level IPPROTO_IP and type IP_TOS.
Apple and FreeBSD accept a cmsg with level IPPROTO_IPV6 and type IPV6_TCLASS.

The same applies to the Linux-specific sendmmsg() call.

### Windows

Windows uses a cmsg with level IPPROTO_IP and type IP_ECN for IPv4 packets.

Windows uses a cmsg with level IPPROTO_IPV6 and type IPV6_ECN for IPv6 packets.

An example of the technique described above can be found at
{{CHROMIUM-WINDOWS}}.

# Security Considerations

The security implications of ECN are documented in {{RFC3168}} and {{RFC9330}}.
This document is a guide to enabling these capabilities, which incurs no
additional security considerations.

Note that implementing ECN capabilities on some platforms, but not others, can
help peers identify the operating system in use by a host, which can have
privacy implications. This document aims to mitigate that possibility.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

The author would like to thank Ryan Hamilton, who provided constant advice
through this effort. Randall Meyer from Apple and Nick Grifka from Microsoft
provided useful hints about the behavior of their respective operating systems.
However, the author takes full responsibility for any errors above.

Neal Cardwell, Gorry Fairhurst, Max Franke, Rodney Grimes, Will Hawkins,
Guillaume H&eacute;tier, Max Inden, Jonathan Lennox, Colin Perkins, Marten
Seemann, Michael Tuexen, and Greg White made improvements to this draft.
