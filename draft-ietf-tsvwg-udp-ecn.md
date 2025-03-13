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
  github: "martinduke/udp-ecn"
  latest: "https://martinduke.github.io/udp-ecn/draft-duke-tsvwg-udp-ecn.html"

author:
 -
    fullname: Martin Duke
    organization: Google
    email: martin.h.duke@gmail.com

informative:
 CHROMIUM:
   title: "The Chromium Projects"
   target: "https://www.chromium.org/chromium-projects/"

 CHROMIUM-POSIX:
   title: "udp_socket_posix.cc"
   target: "https://source.chromium.org/chromium/chromium/src/+/main:net/socket/udp_socket_posix.cc"

 CHROMIUM-WINDOWS:
   title: "udp_socket_win.cc"
   target: "https://source.chromium.org/chromium/chromium/src/+/main:net/socket/udp_socket_win.cc"

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

{{?RFC3168}} reserves two bits in the IP header for Explicit Congestion
Notification (ECN), which provides network feedback to endpoint congestion
controllers. This has historically mostly been relevant to TCP ({{?RFC9293}}),
where any incoming ECN codepoints are internally consumed by the kernel, and
therefore imply no application interface except enabling and disabling the
capability.

The Stream Control Transport Protocol (SCTP) ({{?RFC9260}}) has long supported
ECN in its design. SCTP is sometimes carried over DTLS and UDP ({{?RFC8261}}).
In principle, user-space implementers might have leveraged UDP ECN APIs to
deliver ECN codepoints between SCTP and the UDP socket. The author is not aware
of any such efforts.

{{?RFC6679}} defines ECN over RTP over UDP. The author is aware of a research
implementation, but cannot confirm any commercial deployments.

However, QUIC {{?RFC9000}} runs over UDP and has seen wider deployment than
SCTP. The Low Latency, Low Loss, Scalable Throughput (L4S) experiment
({{?RFC9330}}) and QUIC have combined to increase interest in ECN over UDP.

The Chromium Projects ({{CHROMIUM}}) provide a widely-deployed protocol library
that includes QUIC. An effort to provide ECN support for QUIC on the many
platforms on which Chromium is deployed revealed that many ECN-related UDP
socket interfaces are poorly documented.

This informational document provides a record of that experience, to encourage
further support for ECN in other QUIC implementations, and indeed any consumer
of ECN codepoints that operates over UDP. It is not a standards-track document
and does not bind platforms to any API, or suggest any such API.

Many socket APIs continue to reference the "ToS (Type of Service) byte" even
though {{?RFC2474}} obsoleted that 25 years ago. That 8-bit field now contains a
6-bit Differentiated Services Code Point (DSCP), in addition to the ECN bits.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document is not a general tutorial on UDP socket programming, and assumes
familiarity with basic socket concepts like binding, socket options, and
common system error codes.

# Receiving ECN codepoints

Network devices can change the ECN codepoint in the IP header. Since this
feedback is required at the packet sender, the packet receiver needs to extract
this codepoint from the UDP socket in order to report to the sender.

There are two components to this: setting the socket to report incoming ECN
marks, and retrieving the value for each incoming packet.

## Setting the socket to report incoming ECN codepoints

### Linux, Apple and FreeBSD

To report ECN, applications set a socket option to true using a setsockopt()
call.

IPv6 sockets require a socket option of level IPPROTO_IPV6 and name
IPV6_RECVTCLASS.

IPv4 sockets require a socket option of level IPPROTO_IP and name
IP_RECVTOS.

For dual-stack sockets, on Linux hosts the application sets both the
IPV6_RECVTCLASS and IP_RECVTOS options to receive ECN codepoints on all incoming
packets. On Apple and FreeBSD hosts, the application only sets the
IPPROTO_IPV6-level socket option with name IPV6_RECVTCLASS to receive codepoints
for both v4 and v6; setting an IPPROTO_IP-level socket option on an IPv6 socket
results in an error. In particular this applies to the IPPROTO_IP-level socket
option with the name IP_RECVTOS.

At the time of writing, an example implementation can be found at
{{CHROMIUM-POSIX}}.

### Windows

Windows documentation recommends using the function WSASetRecvIPEcn() to
enable ECN reporting regardless of the IP version.

However, this can also be accomplished by calling setsockopt() and using
options of level IPPROTO_IP and name IP_RECVECN for IPv4, and IPPROTO_IPV6
and IPV6_RECVECN for IPv6. These options are documented at
{{WINDOWS-SOCKOPT}}.

For dual-stack sockets, WSASetRecvIPEcn() will not enable ECN reporting for
IPv4. This requires a separate setsockopt() call using the IP_RECVECN option.

If a socket is bound to a IPv6-mapped IPv4 address (i.e. it is of the format
::ffff:&lt;IPv4 address&gt;), calls to WSASetRecvIpEcn() return error EINVAL.
These sockets should instead use an explicit setsockopt() call to set
IP_RECVECN.

At the time of writing, an example implementation can be found at
{{CHROMIUM-WINDOWS}}.

## Retrieving ECN codepoints on incoming packets

All platforms described in this document require the use of a recvmsg() call to
read data from the socket to retrieve ECN information, because that information
is encoded in the control data that is returned from that function. Those
platforms all return zero or more "cmsg" that contain requested information
about the arriving packet.

Examples of the technique described below can be found at {{CHROMIUM-POSIX}}
and {{CHROMIUM-WINDOWS}}.

### Linux

If the incoming packet is IPv4, Linux will include a cmsg of level IPPROTO_IP
and type IP_TOS.

If the incoming packet is IPv6, Linux will include a cmsg of level IPPROTO_IPV6
and type IPV6_TCLASS.

The resulting report contains the entire IP header byte, which includes other
fields. The ECN codepoint constitutes the two least-significant bits of this
byte.

The same applies to the Linux-specific recvmmsg() call.

### Apple and FreeBSD

If a UDP message (UDP/IPv4) is received on an IPv4 socket, the ancillary data
will contain a cmsg of level IPPROTO_IP and type IP_RECVTOS.
The cmsg data contains an unsigned char.

If a UDP message (UDP/IPv6 or UDP/IPv4) is received on an IPv6 socket, the
ancillary data will contain a cmsg of level IPPROTO_IPV6 and type IPV6_TCLASS.
The cmsg data contains an int.

The provided data is the entire byte from the IP header, which includes other
fields. The ECN codepoint constitutes the two least-significant bits of this
byte.

### Windows

If the incoming packet is IPv4, the socket will include a cmsg of level
IPPROTO_IP and type IP_ECN.

If the incoming packet is IPv6, the socket will include a cmsg of level
IPPROTO_IPV6 and type IPV6_ECN.

The resulting integer solely consists of the ECN codepoint, and requires no
further bitwise operations.

# Sending ECN codepoints

Existing ECN specifications envision a particular connection consistently
sending the same ECN codepoint. It might transition that marking after
successfully completing a handshake, recognizing the path or the peer do not
support ECN, or transitioning to a new path. Therefore, using a socket option
to configure a consistent marking is generally more resource-efficient.

However, some server designs receive all incoming packets on a single socket.
As the many connections that constitute this packet stream may have different
support for ECN, it is suitable to configure outgoing ECN on a per-packet basis.

## On a per-socket basis

### Linux and Apple

Both Linux and Apple platforms set the outgoing ECN for IPv4 packets with a
socket option of level IPPROTO_IP and name IP_TOS.

For IPv6 packets, they use level IPPROTO_IPV6 and name IPV6_TCLASS.

This setsockopt() call also sets the Differentiated Services Code Point (DSCP)
bits that make up the rest of the header byte. Applications making this call will
generally want to preserve any existing DSCP setting, which might require a
getsockopt() call.

For dual-stack sockets, Linux requires an additional setsockopt() call with
IP_TOS. Apple sockets do not and will return an error if this call is made.

An example of the technique described above can be found at {{CHROMIUM-POSIX}}.

### Windows

At the time of this writing, Windows does not provide a way to configure
marking on a per-socket basis.

## On a per-packet basis

Packets can be individually marked with ECN codepoints using the control
information that accompanies a sendmsg() call.

### Linux and Apple

These platforms expect a cmsg with level IPPROTO_IP and type IP_TOS if the
destination is an IPv4 address, or a IPv4-mapped IPv6 address.

Otherwise, they expect a cmsg with level IPPROTO_IPV6 and type IPV6_TCLASS.

The same applies to the Linux-specific sendmmsg() call.

### Microsoft

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
Guillaume H&eacute;tier, Max Inden, Colin Perkins, and Michael Tuexen made
improvements to this draft.
