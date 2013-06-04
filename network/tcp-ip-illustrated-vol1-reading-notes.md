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

### ICMP Error Message

An ICMP error message is not to be sent in response to any of the following
messages: another ICMP error message, datagrams with bad headers(e.g. bad
checksum), IP-layer broadcast/multicast datagrams, datagrams encapsulated in
link-layer broadcast or multicast frames, datagrams with an invalid or network
zero source address, or any fragment other than the first. The reason for
imposing these restrictions on the generation of ICMP errors is to limit the 
creation of so-called broadcast storms, a scenario in which the generation of a
small number of messages creates an unwanted traffic cascade.

An ICMPv4 error message is never generated in response to

* An ICMPv4 error message.(An ICMPv4 error message may, however, be generated
in response to an ICMPv4 query message.)
* A datagram destined for an IPv4 broadcast address or an IPv4 multicast 
address(formerly known as a class D address)
* A datagram sent as a link-layer broadcast
* A fragment other than the first
* A datagram whose source address does not define a single host. This means 
that the source address cannot be a zero address, a loopback address, a
broadcast address, or a multicast address

An ICMPv6 error message is never generated in response to

* An ICMPv6 error message
* An ICMPv6 Redirect message
* A packet destined for an IPv6 multicast address, with two exceptions:
  - The Packet Too Big(PTB) message
  - The Parameter Problem message(code 2)
* A packet sent as a link-layer multicast(with the exceptions noted previously)
* A packet sent as a link-layer broadcast(with the exceptions noted previously)
* A packet whose source address does not uniquely identify a single node. This
means that the source address cannot be an unspecified address, an IPv6 
multicast address, or any address known by the sender to be an anycast address


    % sysctl -a | grep icmp_rate
    net.ipv4.icmp_ratemask = 6168
    net.ipv4.icmp_ratelimit = 100

The ratemask variale indicates which messages have the limit applied to them,
by turning on the kth bit in the mask if the message with code number k is to
be limited, starting from 0. In this case, codes 3, 4, 11 and 12 are being
limit ed (because 6168=0x1818=0001100000011000, where bits 3, 4, 11 and 12
from the right are turned on).

### ICMP Query/Informational Message

#### Echo Request/Reply(ping)

The (DUP!) notation indicates that an Echo Reply has been received containing
a Sequence Number field identical to a previously received one.

#### Multicast Listener Query/Report/Done(ICMPv6 types 130/131/132)

Multicast Listener Discovery(MLD) provides management of multicast addresses
on links using IPv6. It is similar to the IGMP protocol used by IPv4.

### Neighbor Discovery in IPv6

The Neighbor Discovery Protocol in IPv6(sometimes abbreviated as NDP or ND)
brings together the Router Discovery and Redirect mechanisms of ICMPv4 with
the address-mapping capabilities provided by ARP.

#### ICMPv6 Router Solicitation and Advertisement (ICMPv6 Types 133, 134)

Router Advertisement (RA) messages indicate the presence and capabilities of a
nearby router. They are sent periodically by routers, or in response to a
Router Solicitation(RS) message. The RS message is used to induce on-link
routers to send RA messages.

#### ICMPv6 Neighbor Solicitation and Advertisement (ICMPv6 Types 135, 136)

The Neighbor Solicitation(NS) message in ICMPv6 effectively replaces the ARP
request messages used with IPv4. Its primary purpose is to convert IPv6
addresses to link-layer addresses. However, it is also used for detecting
whether nearby nodees can be reached, and they can be reached bidirectionally.

The ICMPv6 Neighbor Advertisement(NA) message serves the purpose of the ARP
response message in IPv4 in addition to helping with neighbor unreachability
detection.

#### ICMPv6 Inverse Neighbor Discovery Solicitation/Advertisement (ICMPv6 types 141, 142)

The Inverse Neighbor Discovery(IND) facility in IPv6 originated from a need to
determine IPv6 addresses given link-layer addresses on Frame Relay networks.
It resembles reverse ARP, a protocol once used with IPv4 networks primarily
for supporting diskless computers. Its main function is to ascertain the
network-layer addresses correspending to a known link-layer address.

#### Neighbor Unreachability Detection(NUD)

One of the important features of ND is to detect when reachability between
two systems on the same link has become lost or asymmetric. This is
accomplished using the Neighbor Unreachability Detection(NUD) algorithm. It is
used to manage the neighbor cache present on each node.

### Attacks involving ICMP

The types of attacks involving ICMP fall primarily into three categories:
floods, bombs, and information disclosure. In essence, floods cause a large
amout of traffic to be generated, leading to an effective DoS attack on one or
more computers. The bomb class(sometimes called nuke class) refers to sending
specially constructed messages that cause IP or ICMP processing to crash or
hang. Information disclosure attacks do not typically cause harm by themselves
but can be used to inform the approaches used by other attack methods to avoid
wasting time or avoid being detected.


## Broadcasting and Local Multicasting(IGMP and MLD)

There are four kinds of IP addresses: unicast, anycast, multicast, and
broadcast. IPv4 may use all of them, and IPv6 uses any except the last form.

Generally, only users applications that use the UDP transport protocol take
advantage of broadcasting and multicasting, where it makes sense for an
application to send a single message to multiple recipients. TCP is a
connection-oriented protocol that implies a connection between two hosts and
one process on each host. TCP can use unicast and anycast addresses, but not
broadcast or multicast addresses.

### Multicast

IP multicasting originated using a design based on the way group addressing
works in link-layer ntworks such as Ethernet. In this approach, each station
selects the group address for which it is willing to accept traffic,
irrespective of sender. This approach is also sometimes called any-source
multicast(ASM) because of the insensitivity to the identity of the sender. As
IP multicasting has evolved, an alternative form that is sensitive to the
identity of the sender called source-specific multicast(SSM) has been developed
that allows end stations to explicitly include or exclude traffic sent to a
multicast group from a particular set of senders.

> SSM brings several important benefits over ASM. Because an SSM channel is
> defined by both a source and a group address, group addresses can be re-used
> by multiple sources while keeping channels unique. For instance, the SSM
> channel (192.168.45.7, 232.7.8.9) is different than (192.168.3.104,
> 232.7.8.9), and hosts subscribed to one will not receive traffic from the
> other. This allows for greater flexibility in choosing a multicast group
> while also protecting against denial of service attacks; hosts will only
> receive traffic from explicitly requested sources.
>
> One of the biggest advantages SSM holds over ASM is that it does not rely
> on the designation of a rendezvous point (RP) to establish a multicast
> tree. Because the source of an SSM channel is always known in advance,
> multicast trees are efficiently built from channel hosts toward the source
> (based on the unicast routing topology) without the need for an RP to join
> a source and shared multicast tree. The corollary of this, which may be
> undesirable in some multicast implementations, is that the multicast
> source(s) must be learned in advance via some external method (e.g. manual
> configuration).
>
> refs:
>
>   * http://packetlife.net/blog/2010/jul/27/source-specific-multicast-pim-ssm/
>
>   * http://packetlife.net/blog/2008/oct/20/pim-sm-source-versus-shared-trees/

### Host Address Filtering

In a typical switched Ethernet environment, broadcast and multicast frames are
replicated on all segments within a VLAN, along a spanning tree formed among
the switches. Such frames are delivered to the NIC on each host which checks
the correctness of the frame(using the CRC) and makes a decision about whether
to receive the frame and deliver it to the device driver and network stack.

One of the primary motivations behind the development of the multicast
addressing features was to avoid the overhead of broadcasting.

### The Internet Group Management Protocol(IGMP) and Multicast Listener Discovery Protocol(MLD)

When multicast datagrams are to be forwarded over a wide area network or within
an enterprise across multiple subnets, we require that *multicast routing* be
enabled by one or more multicast routers.

*Reverse Path Forwarding*(RPF) check procedure performs a routing lookup on the
source address of an arriving multicast datagram. Only if the outgoing
interface for routing matches the interface on which the datagram arrived is
the datagram forwarded. The RPF check is important for avoiding multicast
loops. Multicast routing is largely separate from conventional unicast routing
provided by IP routers.

Two major protocols are used to allow multicast routers to learn the groups in
which nearby hosts are interested: the Internet Group Management Protocol(IGMP)
used by IPv4 and the Multicast Listener Discovery(MLD) protocol used by IPv6.
Both are used by hosts and routers that support multicasting, and the protocols
are very similar.

> All Hosts multicast address: 224.0.0.1(IGMP)

> All Nodes link-scope multicast address: ff02::1(MLD)

The IGMP message TTL is set to 1, as IGMP messages are not forwarded through
routers.

### IGMP and MLD Snooping

IGMP and MLD manage the flow of IP multicast traffic among routers. To optimize
traffic flow even further, it is possible for layer 2 switches(that would not
ordinarily process layer 3 IGMP or MLD messages) to become aware of whether
certain multicast traffic flows are of interest or not by looking at layer 3
information. This capability is indicated by a switch feature known as
IGMP(MLD) *snooping* and is supported by many switch vendors.

### Attacks Involving IGMP and MLD

Because IGMP and MLD are signaling protocols that control the flow of multicast
traffic, attacks using these protocols primarily are either DoS attacks or
resource utilization attacks.


## User Datagram Protocol(UDP) and IP Fragmentation

UDP is a simple, datagram-oriented, transport-layer protocol that preserves
message boundaries. It does not provide error correction, sequencing, duplicate
elimination, flow control, or congestion control.

Generally, each UDP output operation requested by an application produces
exactly one UDP datagram, which causes one IP datagram to be sent. This is in
contrast to a stream-oriented protocol such as TCP, where the amount of data
written by an application may have little relationship to what actually gets
sent in a single IP datagram or what is consumed at the receiver.

### UDP Header

UDP header is always 8 bytes in size.

    source port number: 2 bytes
    destination port number: 2 bytes
    length: 2 bytes
    checksum: 2 bytes

### UDP Checksum

Transport protocols(e.g. TCP, UDP) use checksums to cover their their headers
and data. With UDP, the checksum is optional(although strongly suggested),
while with the others it is mandatory.

Although the basics for calculating the UDP checksum are similar to the general
Internet checksum, there are two small special details.

* First, the length of the UDP datagram can be an odd number of bytes, whereas
  the checksum algorithm adds 16-bit words(always an even number of bytes). The
  procedure for UDP is to append a (virtual) pad byte of 0 to the end of
  odd-length datagrams, just for the checksum computation and verification.
  This pad byte is not actually transmitted and is thus called "virtual" here.

* The second detail is that UDP(as well as TCP) computes its checksum over a
  12-byte pseudo-header derived(solely) from fields in the IPv4 header or a
  40-byte pseudo-header derived from fields in the IPv6 header. This
  pseudo-header is also virtual and is used only for purposes of the checksum
  compuation(at both the sender and the receiver). It is never actually
  transmitted. This pseudo-header includes the source and destination addresses
  and *Protocol* or *Next Header* field from the IP header.

Despite UDP checksums being optional in the original UDP specification, they
are currently required to be enabled on hosts by default. During the 1980s some
computer vendors turned off UDP checksums by default to speed up their
implementation of Sun's Network File System(NFS), which uses UDP. While this
might not cause problems in many cases because of the presence of layer 2 CRC
protection (which is stronger than the Internet checksum), it is considered bad
form (and a violation of the RFCs) to disable checksums by default. Early
experience in the Internet revealed that when datagrams pass through routers,
all bets are off with respect to their correctness. Believe it or not, there
have been routers with software and hardware bugs that have modified bits in
the datagrams being forwarded. These errors are undetectable in a UDP datagram
if the end-to-end UDP checksum is disabled. Also realize that some older
data-link protocols (e.g., serial line IP, or SLIP) do not have any form of
data-link checksum, thereby leaving open the possibility that IP packets could
be undetectably modified unless another checksum is employed.

Given the structure of the pseudo-header, it is clear that when a UDP/IPv4
datagram passes through a NAT, not only is the IP-layer header checksum
modified, but the UDP pseudo-header checksum must be appropriately modified
because the IP-layer addressing and/or UDP-layer port numbers may have changed.
NATs therefore routinely perform “layering violations” by modifying multiple
layers of protocol within packets at the same time. Of course, given that the
pseudo-header is itself a layering violation, a NAT has little choice.

### UDP and IPv6

In IPv6, the minimum MTU size is 1280 bytes(as opposed to the 576 bytes
required by IPv4 as the minimum size required to be supported by all hosts).

IPv6 supports jumbograms(packets larger than 65535 bytes). If we inspect the
IPv6 header and option set, we can observe that with jumbograms, 32 bits are
available to hold the payload length. This implies that a single UDP/IPv6
datagram could be very large indeed.

### Teredo: Tunneling IPv6 through IPv4 Networks

Although it was once thought that a worldwide transition to IPv6 might happen
quickly, this has not materialized exactly as forecast. Consequently, a number
of (theoretically temporary) transition mechanisms have been proposed to ease
the transition burden. One such machanism is called 6to4, whereby IPv6 packets
used by hosts are encapsulated in IPv4 packets that may be delivered over an
IPv4-only infrastructure.

Teredo transports IPv6 datagrams in the payload area of UDP/IPv4 datagrams for
systems that have no other IPv6 connectivity options.

### IP Fragmentation

To keep the IP datagram abstraction consistent and isolated from link-layer
details, IP employs *fragmentation* and *reassembly*.

Fragmentation in IPv4 can take place at the original sending host and at any
intermediate routers along the end-to-end path. Note that datagram fragments
can themselves be fragmented. Fragmentation in IPv6is somewhat different
because only the source is permitted to perform fragmentation.

When an IP datagram is fragmented, it is not reassembled until it reaches its
final destination. Two reasons have been given for this. First, not performing
reassembly within the network alleviates the forwarding software (or hardware)
in routers from implementing this feature. Second, it is possible for different
fragments of the same datagram to follow different paths to their common
destination. If any fragment is lost, the entire datagram is lost.

### Path MTU Discovery with UDP

For a protocol such as UDP, in which the calling application is generally in
control of the outgoing datagram size, it is useful if there is some way to
determine an appropriate datagram size if gragmentation is to be avoided.
Conventional PMTUD uses ICMP PTB messages in determining the largest packet
size along a routing path that can be used without inducing fragmentation.

### Lack of Flow and Congestion Control

UDP provides no flow control(that is, no way for the server to tell the client
to tell the client to slow down).

UDP poses a special concern for congestion because it has no way of being
informed that it should slow down its sending rate if the network is being
driven into congestion. (It also has no mechanism for slowing down, even if it
were told to do so.). Thus, it is said to lack congestion control.

### Attacks Involving UDP and IP Fragmentation

The most straightforward DoS attack with UDP is simply generating massive
amounts of traffic as fast as possible. Because UDP does not regulate its
sending traffic rate, this can negatively impact the performance of other
applications sharing the same network path.


## Name Resolution and the Domain Name System(DNS)

### The DNS Name Space

The set of all names used with DNS constitutes the DNS name space. The current
DNS name space is a tree of domains with an unamed root at the top. The top
echelons of the tree are the so-called top-level domains(TLDs), which include
generic TLDs(gTLDs), country-code TLDs(ccTLDs), and internationalized
country-code TLDs(IDN ccTLDs), plus a special infrastructure TLD called, for
historical reasons, ARPA.

### DNS Naming Syntax

The names below a TLD in the DNS name tree are further partitioned into groups
known as subdomains.

Fully Qualified Domain Names(FQDNs) are sometimes written more formally with a
trainling period(e.g., mit.edu.). This trailing period indicates that the name
is complete; no additional information should be added to the name when
performing a name resolution.

### Name Servers and Zones



