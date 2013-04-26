# TCP/IP Illustrated, vol.1 reading notes

## vlan 802.1q

tcpdump 'vlan' pcap filter will shift 4 bytes right, be careful
when filtering both vlan tagged and untagged packets.

"tcpdump -d" could dump the raw bpf filter in a human readable way. 

> refs: http://www.christian-rossow.de/articles/tcpdump_filter_mixed_tagged_and_untagged_VLAN_traffic.php


## bonding 802.1ax(formerly 802.3ad)

LACP(link aggregation control protocol) supplies several algorithms:
RR, HA, Master-Slave, etc.


## STP

RSTP doesn't use special TC(topology change) BPDUs which original STP use.

RSTP sends periodical "keepalives" BPDUs to determine if connections are
working properly.


## Data Fragmentation

one reason of data fragmentation is that there is probability that data cannot
be transfer successfully.

if there is no data fragmentation, the cost of re-transmit is too high.


## Point-to-Point Protocol(PPP)

PPP is a popular method to carry IP datagrams over serial links and should be
considered a collection of protocols.

Link Control Protocol(LCP) is used to establish and maintain a low-level
two-party communication

NOTE: With Ethernet, up to 16 retransmission attempts are made before giving
up. But PPP is configured to do no restransmission typically.

PPP auth: PAP(raw), CHAP( vulnerable to man in the middle ), EAP(an auth
framework that can carry many auth methods like PAP, CHAP, etc. used mostly by
enterprise and ISP).

Network Control Protocol(NCP) is used to establish network layer communication.
For IPv4, the NCP is IP Control Protocol(IPCP). For IPv6, the NCP is IPV6CP.


## Tunneling

Tunneling is the idea of carrying lower-layer traffic in higher-layer packets.

pptpd acts as a GRE tunneling agent and relays packets to pppd.

link operates in only one direction: unidirectional link(UDL).

promiscuous mode: allow interfaces to receive traffic even if it is not
destined for them.


## ARP

Proxy ARP lets a system answer ARP requests for a different host.
Alias: promiscuous ARP, ARP hack.

linux: echo 1 > /proc/sys/net/ipv4/*/proxy_arp to enable auto-proxy ARP.

Gratuitous ARP occurs when a host sends an ARP request looking for its own
address.

Two goals:
1. determine if another host is already configured with same ipv4 address.
2. let other hosts update the arp cache in case there is a hardware address
change.

Address Conflict Detection(ACD) defines ARP probe and ARP annoucement packets.

ARP filter: /proc/sys/net/ipv4/conf/*/arp_filter


## The Internet Protocol(IP)

IP provides a best-effort(no guarantee), connectionless(no connection state)
datagram delivery service.

"tcpdump -xx" could dump packets with ethernet header

### ethernet header

    octet: content
    0-2: dst mac addr
    3-5: src mac addr
    (4 octets): vlan tag
    6-7: ethertype, 0x0800(ipv4)/0x0806(arp)/0x8100(vlan)/0x86dd(ipv6)
    #followed by ethernet payload

### ipv4 header

    0: version(0-3 bit), 4(ipv4)/6(ipv6)
       IHL(Internet Header Length)(4-7 bit), header length in octet,
       minimum is 5(indicates 20 bytes).
    1: DSCP(Differentiated Services Code Point)(8-13 bit), former ToS.
       ECN( Explicit Congestion Notification )(14-15 bit), don't care...
    2-3: Total Length, packet(fragment) size, including the header
    4-5: Identification
    6-7: Flags(16-18 bit), bit 0: Researved, must be zero;
                           bit 1: Don't Fragment(DF), bit 2: More Fragment(MF)
         Fragment Offset(19-31 bit)
    8: TTL(Time To Live)
    9: Protocol, 0x01(icmp)/0x02(igmp)/0x06(tcp)/0x11(udp)
    10-11: header checksum, CRC
    12-15: src ip addr
    16-19: dst ip addr

ipv6 header is 40 bytes fixed size and extend headers are added when necessary.
Next Header field in each header indicates the type of subsequent header.

IPv6 Jumbo Payload option provides a 32-bit field for holding the payload size
for datagrams with size between 65535 and 2^32 bytes.

There are two host models: the strong host model and the weak host model.

In strong host model, a datagram is accepted for delivery to the local protocol
stack only if the ip address contained in the dst ip addr field matches one of
those configured on the interface upon which the datagram arrived.

The weak host model vice versa.

Windows(Vista and later) uses the strong model host model by default whereas
Linux defaults to the weak model.


## Dynamic Host Configuration Protocol(DHCP)

The design of DHCP is based on an earlier protocol callled the
Internet Bootstrap Protocol(BOOTP).

BOOTP and DHCP are carried using UDP/IP. Clients use port 68 and servers use port 67.

### DHCPv4

    Client | Server

    --> DISCOVER(null ciaddr; broadcast; xid; params request)
    <-- OFFER(broadcast; siaddr; xid; yiaddr; options)(maybe more than one server)
    --> REQUEST(broadcast; siaddr; ciaddr; xid; options)
      <-- ACK(broadcast; xid; yiaddr; options)
      (Verifying, e.g. ACD; recommended; DECLINE if conflict)

### DHCPv6

    Client | Server
    --> SOLICIT(null/mcast; xid; DUID, options)
    <-- ADVERTISE(mcast; xid, ip addr, server id, options)
    --> REQUEST(mcast; ip addr, server id, xid)
    <-- REPLY(mcast; xid, options)
    (Verifying, e.g. DAD; recommended; DECLINE if conflict)

The approximately equivalent DHCPv4 messages for DHCPv6:

    DHCPv6  |  DHCPv4
    SOLICT     DISCOVER
    ADVERTISE  OFFER
    REQUEST    REQUEST
    CONFIRM    REQUEST
    RENEW      REQUEST
    REBIND     DISCOVER
    REPLY      ACK/NAK
    RELEASE    RELEASE
    DECLINE    DECLINE
    RECONFIGURE FORCERENEW
    INFORMATION-REQUEST INFORM
    RELAY-FORW N/A
    RELAY-REPL N/A
    LEASEQUERY LEASEQUERY
    LEASEQUERY-REPLY LEASE{UNASSIGNED, UNKNOW, ACTIVE}
    LEASEQUERY-DONE LEASEQUERYDONE
    LEASEQUERY-DATA N/A
    N/A BULKLEASEQUERY

IPv6 Duplicate Address Detection(DAD) uses
ICMPv6 Neighbor Solicitation and Neighbor Advertisement messages to determine
if a particular (tentative or optimistic) IPv6 address is already in use on
the attached link.


## Network Address Translation(NAT)

Two major types of firewalls include proxy firewalls and packet-filtering
firewalls.

Modify an address at the ip layer also requires modifying checksums at the
transport layer.

Network Address Port Translation(NAPT) uses the transport-layer identifiers
(i.e. ports for tcp and udp, query identifiers for icmp) to differentiate which
host on the private side of the NAT is associated with a particular packet.

Internet Control Message Protocol(ICMP) provides status information about ip
packets and can also be used for making certain measurements and gathering
information about the state of the network.

ICMP has two categories of messages: informational and error.

Error messages generally contain a (partial or full) copy of the ip packet that
induced the error condition. They are sent from the point where error was
detected, often in the middle of the network, to the original sender.

Ordinarily, this presents no difficulty, but when an ICMP error message passes
through a NAT, the ip addr in the included "offending datagram" need to be
rewritten by the NAT in order for them to make sense to the end client
(called ICMP fix-up).

In IPv6, other desirable NAT features(e.g., firewall-like functionality,
topology hiding, and privacy) can be better achieved using
Local Network Protection(LNP). LNP represents a collection of techniques
with IPv6 that match or exceed the properties of NATs.

Three NAT types: endpoint-independent, address-dependent, address-dependent
and port-dependent.

### NAT hairpin

A client wishes to reach a server and both reside on the same, private side of
the same NAT. NATs that support this scenario implement so-called hairpinning
or NAT loopback.

### hole punching

A method that attempts to allow two or more systems, each behind a NAT, to
communicate directly using pinholes is called hole punching. To punch a hole,
a client contacts a known server using an outgoing connection that establishes
a mapping in its local NAT. When another client contacts the same server, 
the server has connections to each of the clients and knows their external
addressing information between the clients. Once this information is know,
a client can attempt a direct connection to the other client.

### UNilateral Self-Address Fixing(UNSAF)

Applications employ a number of methods to determine the addresses their
traffic will use when passed through a NAT.
This is called fixing(learning and maintaining) the addressing information.

One of the primary workhorses for UNSAF and NAT traversal is called
Session Traversal Utilities for NAT (STUN).
STUN has evolved from a previous version called Simple Tunneling of UDP through
NATs, now known as "classic STUN".


## Internet Control Message Protocol(ICMP)

A special protocol called the Internet Control Message Protocol(ICMP) is used
in conjunction with IP to provide diagnostics and control information related
to the configuration of the IP protocol layer and the disposition of IP
packets.

Note that ICMP does not provide reliability for IP. Rather, it indicates
certain classes of failures and configuration information. The most common
cause of packet drops (buffer overrun at a router) does not elicit any ICMP
information.

Because of the ability of ICMP to affect the operation of import system
functions and obtain configuration information, hackers have used ICMP messages
in a large number of attacks. As a result of concerns about such attacks,
network administrators often arrange to block ICMP messages with firewalls,
especially at border routers.

ICMP messages are encapsulated for transmission within IP datagrams.

### ICMPv4 Messages

    type | name
    0      Echo Reply
    3      Destination Unreachable
    5      Redirect
    8      Echo Request
    11     Time Exceeded(e.g. IPv4 TTL)
    12     Parameter Problem(Malformed packet or header)

### ICMPv6 Messages

    type | name
    1      Destination Unreachable
    3      Time Exceeded
    4      Parameter Problem
    128    Echo Request
    129    Echo Reply
    137    Redirect Message

Following rules are applied when processing incoming ICMPv6 messages:

1. Unknown ICMPv6 error messages must be passed to the upper layer process
that produced the datagram causing the error(if possible).
2. Unknown ICMPv6 informational messages are dropped.
3. ICMPv6 error messages include as much of the original ("offending") IPv6
datagram that caused the error as will fit without making the error message
datagram exceed the mininum IPv6 MTU(1280 bytes).
4. When processing ICMPv6 error messages, the upper-layer protocol type is
extracted from the original or "offending" packet(contained in the body of
the ICMPv6 error message) and used to select the appropriate upper layer
process. If this is not possible, the error message is silently dropped after
any IPv6-layer processing.
5. There are special rules for handling errors(see Section 8.3).
6. An IPv6 node must limit the rate of ICMPv6 error messages it sends. There
are a variety of ways of implementing the rate-limiting function, including the
token bucket approach mentioned in Section 8.3.


