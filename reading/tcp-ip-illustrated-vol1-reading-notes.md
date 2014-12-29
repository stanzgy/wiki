# TCP/IP Illustrated, vol.1 reading notes

## vlan 802.1q

tcpdump 'vlan' pcap filter will shift 4 bytes right, be careful
when filtering both vlan tagged and untagged packets.

"tcpdump -d" could dump the raw bpf filter in a human readable way. 

> refs:
>
>   [1] http://www.christian-rossow.de/articles/tcpdump_filter_mixed_tagged_and_untagged_VLAN_traffic.php


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

The unit of administrative delegation, in the language of DNS servers, is
called a zone. A zone is a subtree of the DNS name space that can be
administered separately from other zones. Every domain name exists within some
zone, even the TLDs that exist inthe root zone.

### Caching

Most name servers also cache zone information they learn, up to a time limit
called the *time to live(TTL)*.


## TCP: The Transmission Control Protocol(Preliminaries)

### Windows of Packets and Sliding Windows

We define a window of packets as the collection of packets(or their sequence
numbers) that have been injected by the sender but not yet completely
acknowledged(i.e., the sender has not received an ACK for them).

We refer to the window size as the number of packets in the window.

The movement of the window gives rise to another name for this type of
protocol, a *sliding window* protocol.

### Variable Windows: Flow Control and Congestion Control

To handle the problem that arises when a reveiver is too slow relative to a
sender, we introduce a way to force the sender to slow down when the receiver
cannot keep up. This is called flow control and is usually handled in one of
two ways. One way, called rate-based flow control, gives the sender a certain
data rate allocation and ensures that data is never allowed to be sent at a
rate that exceeds the allocation. This type of flow control is most appropriate
for streaming applications and can be used with broadcast and multicast
delivery.

The other predominant form of flow control is called window-based flow control
and is the most popular approach when sliding windows are being used. In this
approach, the window size is not fixed but is instead allowed to vary over
time. To achieve flow control using this tachnique, there must be a method for
the receiver to signal the sender how large a window to use. This is typically
called a *window advertisement*, or simply a *window update*. This value is
used by the sender to adjust its window size. Logically, a window update is
separate from the ACKs we discussed previously, but in practice the window
update and ACK are carried in a single packet, meaning that the sender tends to
adjust the size of its window at the same time it slides it to the right.

We may have routers with limited memory between the sender and the receiver
that have to contend with slow network links. When this happens, it is possible
for the sender's rate to exceed a router's ability to keep up, leading to
packet loss. This is addressed with a special form of flow control called
congestion control.

### Setting the Retransmission Timeout

Because it is not practical for the user to tell the protocol implementation
what the values of all the times are(or to keep them up-to-date) for all
circumstances, a better strategy is to have the protocol implementation try to
estimate them. This is called *round-trip-time estimation* and is a statistical
process.

### Introduction to TCP

#### The TCP Service Model

TCP provides a *connection-oriented*, reliable, byte stream service. The term
*connection-oriented* means that the two applications using TCP must establish
a TCP connection by contacting each other before they can exchange data.

#### Reliability in TCP

The application data is broken into what TCP considers the best-size chunks to
send, typically fitting each segment into a single IP-layer datagram that will
not be fragmented. This is different from UDP, where each write by the
application usually generates a UDP datagram of that size(plus headers). The
chunk passed by TCP to IP is called a *segment*.

The ACKs used by TCP are *cumulative* in the sense that an ACK indicating byte
number N implies that all bytes up to number N(but not including it) have
already been received sucessfully.

TCP provides a *full-duplex* service to the application layer. This means that
data can be flowing in each direction, independent of the other direction.

Using sequence numbers, a receiving TCP discards duplicate segments and
reorders segments that arrive out of order. Because it is a byte stream
protocol, however, TCP never delivers data to the receiving application out of
order.

### TCP Header and Encapsulation

When a new connection is being established, the SYN bit field is turned on in
the first segment sent from client to server. Such segments are called SYN
segments, or simply SYNs. The *Sequence Number* field then contains the first
sequence number to be used on that direction of the connection for subsequent
sequence numbers and in returning ACK numbers(recall that connections are all
bidirectional). Note that this number is not 0 or 1 but instead is another
number, often randomly chosen, called the *initial sequence number*(ISN). The
sequence number of the first byte of data sent on this direction of the
connection is the ISN plus 1 because the SYN bit field consumes one sequence
number.

Modern TCPs, have a *selective acknowledgment*(SACK) option that allows the
receiver to indicate to the sender out-of-order data it has received correctly.

Currently eight bit fields are defined for the TCP header. One or more of them
can be turned on at the same time.

* CWR Congestion Window Reduced
* ECE ECN Echo
* URG Urgent
* ACK Acknowledgement
* PSH Push
* RST Reset the connection
* SYN Synchronize sequence numbers
* FIN The sender of the segment is finished sending data to its peer

The most common *Option* field is the Maximum Segment Size option, called the
MSS. Each end of a connection normally specifies this option on the first
segment it sends(the ones with the SYN bit field set to establish the
connection). The MSS option specifies the maximum-size segment that the sender
of the option is willing to receive in the reverse direction.

## TCP Connection Management

### Introduction

TCP is a unicast *connection-oriented* protocol. TCP's service model is a byte
stream.

Because of its management of *connection state* (information about the
connection kept by both endpoints), TCP is a considerably more complicated
protocol than UDP. UDP is a *connectionless* protocol that involves no
connection establishment or termination.

### TCP Connection Establishment and Termination

A TCP connection is defined to be a 4-tuple consisting of two IP addresses and
two port numbers. More precisely, it is a pair of endpoints or sockets where
each endpoint is identified by an (IP address, port number) pair.

    Active Opener(Client) | Passive Opener(Server)
    ---> SYN, Seq=ISN(c), (options)
    <--- SYN+ACK, Seq=ISN(s), ACK=ISN(c)+1, (options)
    ---> ACK, Seq=ISN(c)+1, ACK=ISN(s)+1, (options)

    ---> FIN+ACK, Seq=K, ACK=L, (options)
    <--- ACK, Seq=L, ACK=K+1, (options)
    <--- FIN+ACK, Seq=L, ACK=K+1, (options)
    ---> ACK, Seq=K, ACK=L+1, (options)

The half-close operation in TCP closes only a single direction of the data
flow. Two half-close operations together close the entire connection. The
sending of a FIN is normally the result of the application issueing a close
operation, which typically causes both directions to close.

#### TCP Half-Close

The Berkeley sockets API supports half-close, if the application calls the
shutdown() function instead of calling the more typical close() function.

    Active Opener(Client) | Passive Opener(Server)
    ---> FIN+ACK, Seq=K, ACK=L, (options)
    <--- ACK, Seq=L, ACK=K+1, (options)
    <--- [More Data Sent]
    ---> [Data Acknowledged]
    <--- FIN, Seq=L, ACK=K+1, (options)
    ---> ACK, Seq=K, ACK=L+1, (options)

#### Simultaneous Open and Close

It's possible for two applications to perform an active open to each other at
the same time. Each end must have transmitted a SYN before receiving a SYN from
the other side; the SYNs must pass each other on the network. This scenario
also requires each end to have an IP address and port number that are known to
the other end, which is rare. If this happens, it is called a simultaneous
open.

#### Initial Sequence Number(ISN)

RFC0793 specifies that the ISN should be viewed as a 32-bit counter that
increments by 1 every 4us. The purpose of doing this is to arrange for the
sequence numbers for segments on one connection to not overlap with sequence
numbers on a another (new) identical connection.

#### Connections and Translators

When NAT is used with TCP, the pseudo-header checksum usually requires
adjustment. This is also true for other protocols that use pseudo-header
checksums.

### TCP Options

Every option begins with a 1-byte kind that specifies the type of option.
Options that are not understood are simply ignored.

TCP header's length is always required to be a multiple of 32 bits because the
TCP Header Length field uses that unit.

#### Maximum Segment Size (MSS) Option

The maximum segment size (MSS) is the largest segment that a TCP is willing to
receive from its peer and, consequently, the largest size its peer should ever
use when sending. The MSS value counts only TCP data bytes and does not include
the sizes of any associated TCP or IP header. When a connection is established,
each end usually announces its MSS in an MSS option carried with its SYN
segment.

#### Selective Acknowledgment (SACK) Options

Because TCP uses cumulative ACKs, TCP is never able to acknowledge data it has
received correctly but that is not contiguous, in terms of sequence numbers,
with data it has received previously. In such cases, the TCP receiver is said
to have holes in its received data queue.

A TCP learns that its peer is capable of advertising SACK information by
receiving the SACK-Permitted option in a SYN (or SYN + ACK) segment.

SACK information contained in a SACK option consists of a range of sequence
numbers representing data blocks the receiver has successfully received. Each
range is called a SACK block and is represented by a pair of 32-bit sequence
numbers. Thus, a SACK option containing n SACK blocks is (8n + 2) bytes long.
Two bytes are used to hold the kind and length of the SACK option.

Although the SACK-Permitted option is only ever sent in a SYN segment, the
SACK blocks themselves may be sent in any segment once the sender has sent the
SACK-Permitted option.

#### Window Scale (WSCALE or WSOPT) Option

The Window Scale option (denoted WSCALE or WSOPT) [RFC1323] effectively
increases the capacity of the TCP Window Advertisement field from 16 to about
30 bits. Instead of changing the field size, however, the header still holds a
16-bit value, and an option is defined that applies a scaling factor to the
16-bit value. This, in effect, multiplies the window value by the value 2s,
where s is the scale factor. The 1-byte shift count is between 0 and 14
(inclusive). A shift count of 0 indicates no scaling. The maximum scale value
of 14 provides for a maximum window of 1,073,725,440 bytes (65,535 × 214),
close to 1,073,741,823 (230 −1), effectively 1GB. TCP then maintains the “real”
window size internally as a 32-bit value.

This option can appear only in a SYN segment, so the scale factor is fixed in
each direction when the connection is established.

#### Timestamps Option and Protection against Wrapped Sequence Numbers (PAWS)

The Timestamps option (sometimes called the Timestamp option and written as
TSOPT or TSopt) lets the sender place two 4-byte timestamp values in every
segment. The receiver reflects these values in the acknowledgment, allowing the
sender to calculate an estimate of the connection’s RTT for each ACK received.

When using the Timestamps option, the sender places a 32-bit value in the
Timestamp Value field (called TSV or TSval) in the first part of the TSOPT, and
the receiver echoes this back unchanged in the second Timestamp Echo Retry
field (called TSER or TSecr). TCP headers containing this option increase by 10
bytes (8 bytes for the two timestamp values and 2 to indicate the option value
and length).

The PAWS algorithm does not require any form of time synchronization between
the sender and the receiver. All the receiver needs is for the timestamp values
to be monotonically increasing, and to increase by at least 1 per window of
data.

#### User Timeout (UTO) Option

The UTO value (also called USER_TIMEOUT) specifies the amount of time a TCP
sender is willing to wait for an ACK of outstanding data before concluding
that the remote end has failed. USER_TIMEOUT has traditionally been a local
configuration parameter for TCP.

#### Authentication Option (TCP-AO)

TCP Authentication Option (TCP-AO) uses a cryptographic hash algorithm (see
Chapter 18), in combination with a secret value known to each end of a TCP
connection, to authenticate each segment.

This option is intended as a strong countermeasure to a variety of TCP spoofing
attacks. However, because it requires creation and distribution of a shared key
(and is a relatively new option), it is not yet widely deployed.

### Path MTU Discovery with TCP

Knowing the path MTU can help protocols such as TCP avoid fragmentation.
Discovery of the path MTU (PMTUD) is accomplished based on ICMP messages, but
in that case UDP is not usually able to adapt its datagram size because the
application specifies the size (i.e., not the transport protocol).

When a connection is established, TCP uses the minimum of the MTU of the
outgoing interface, or the MSS announced by the other end, as the basis for
selecting its send maximum segment size (SMSS). PMTUD does not allow TCP to
exceed the MSS announced by the other end.

There are a number of problems with PMTUD when it operates in an Internet
environment with firewalls that block PTB messages [RFC2923]. Of the various
operational problems with PMTUD, black holes have been the most problematic.

### TCP State Transitions

#### TCP State Transition Diagram

read the diagram

#### TIME_WAIT (2MSL Wait) State

The TIME_WAIT state is also called the 2MSL wait state. It is a state in which
TCP waits for a time equal to twice the Maximum Segment Lifetime (MSL),
sometimes called timed wait. Every implementation must choose a value for the
MSL. It is the maximum amount of time any segment can exist in the network
before being discarded.

When TCP performs an active close and sends the final ACK, that connection must
stay in the TIME_WAIT state for twice the MSL. This lets TCP resend the final
ACK in case it is lost. The final ACK is resent not because the TCP retransmits
ACKs (they do not consume sequence numbers and are not retransmitted by TCP),
but because the other side will retransmit its FIN (which does consume a
sequence number). Indeed, TCP will always retransmit FINs until it receives a
final ACK.

Another effect of this 2MSL wait state is that while the TCP implementation
waits, the endpoints defining that connection (client IP address, client port
number, server IP address, and server port number) cannot be reused. That
connection can be reused only when the 2MSL wait is over, or when a new
connection uses an ISN that exceeds the highest sequence number used on the
previous instantiation of the connection [RFC1122], or if the use of the
Timestamps option allows the disambiguation of segments from a previous
connection instantiation to not otherwise be confused [RFC6191].

Most implementations and APIs provide a way to bypass this restriction. With
the Berkeley sockets API, the SO_REUSEADDR socket option enables the bypass
operation.

For interactive applications, it is normally the client that does the active
close and enters the TIME_WAIT state. The server usually does the passive close
and does not go through the TIME_WAIT state.

With servers, however, the situation is different. They almost always use well-
known ports. If we terminate a server process that has a connection established
and immediately try to restart it, the server cannot assign its assigned port
number to its endpoint (it gets an “Address already in use” binding error),
because that port number is part of a connection that is in a 2MSL wait state.

> refs:
>
>   [1] http://vincent.bernat.im/en/blog/2014-tcp-time-wait-state-linux.html

#### Quiet Time Concept

[RFC0793] states that TCP should wait an amount of time equal to the MSL before
creating any new connections after a reboot or crash. This is called the quiet
time.

#### FIN_WAIT_2 State

In the FIN_WAIT_2 state, TCP has sent a FIN and the other end has acknowledged
it. Unless a half-close is being performed, the TCP must wait for the
application on the other end to recognize that it has received an end-of-file
notification and close its end of the connection, which causes a FIN to be
sent. Only when the application performs this close (and its FIN is received)
does the active closing TCP move from the FIN_WAIT_2 to the TIME_WAIT state.
This means that one end of the connection can remain in this state forever. The
other end is still in the CLOSE_WAIT state and can remain there forever, until
the application decides to issue its close.

Many implementations prevent this infinite wait in the FIN_WAIT_2 state as
follows: If the application that does the active close does a complete close,
not a half-close indicating that it expects to receive data, a timer is set. If
the connection is idle when the timer expires, TCP moves the connection into
the CLOSED state. In Linux, the variable net.ipv4.tcp_fin_timeout can be
adjusted to control the number of seconds to which the timer is set. Its
default value is 60s.


    Normal Close Sequence

    TCP A                                                TCP B

    ESTABLISHED                                          ESTABLISHED

    (Close)
    FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  --> CLOSE-WAIT

    FIN-WAIT-2  <-- <SEQ=300><ACK=101><CTL=ACK>      <-- CLOSE-WAIT

    TIME-WAIT   <-- <SEQ=300><ACK=101><CTL=FIN,ACK>  <-- LAST-ACK

    TIME-WAIT   --> <SEQ=101><ACK=301><CTL=ACK>      --> CLOSED

    (2 MSL)
    CLOSED


##### FIN_WAIT_2 Problem

In order for an HTTP server to reliably implement the protocol it needs to
shutdown each direction of the communication independently. When this feature
was added to Apache it caused a flurry of problems on various versions of Unix
because of a shortsightedness. The TCP specification does not state that the
FIN_WAIT_2 state has a timeout, but it doesn't prohibit it. On systems without
the timeout, Apache 1.2 induces many sockets stuck forever in the FIN_WAIT_2
state.

In many cases this can be avoided by simply upgrading to the latest TCP/IP
patches supplied by the vendor.


Why Does It Happen?

* The client opens a new connection to the same or a different site, which
  causes it to fully close the older connection on that socket.
* The user exits the client, which on some (most?) clients causes the OS to
  fully shutdown the connection.
* The FIN_WAIT_2 times out, on servers that have a timeout for this state.

This is a bug in the browser or in its operating system's TCP implementation.


What Can I Do About it?

* Add a timeout for FIN_WAIT_2
* Compile without using lingering_close() *NOT RECOMMENDED*
* Use SO_LINGER as an alternative to lingering_close()
* Increase the amount of memory used for storing connection state
* Disable KeepAlive

> refs:
>
>   [1] http://httpd.apache.org/docs/current/misc/perf-tuning.html
>
>   [2] http://httpd.apache.org/docs/2.0/misc/fin_wait_2.html


#### Simultaneous Open and Close Transitions

When a simultaneous open occurs, both ends send a SYN at about the same time,
entering the SYN_SENT state. When each end receives its peer’s SYN segments,
the state changes to SYN_RCVD, and each end resends a SYN and acknowledges the
received SYN. When each end receives the SYN plus the ACK, the state changes to
ESTABLISHED.


    Simultaneous Close Sequence

    TCP A                                                TCP B

    ESTABLISHED                                          ESTABLISHED

    (Close)                                              (Close)
    FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  ... FIN-WAIT-1
                <-- <SEQ=300><ACK=100><CTL=FIN,ACK>  <--
                ... <SEQ=100><ACK=300><CTL=FIN,ACK>  -->

    CLOSING     --> <SEQ=101><ACK=301><CTL=ACK>      ... CLOSING
                <-- <SEQ=301><ACK=101><CTL=ACK>      <--
                ... <SEQ=101><ACK=301><CTL=ACK>      -->

    TIME-WAIT                                            TIME-WAIT
    (2 MSL)                                              (2 MSL)
    CLOSED                                               CLOSED


### Reset Segments

In general, a reset is sent by TCP whenever a segment arrives that does not
appear to be correct for the referenced connection.

#### Connection Request to Nonexistent Port

A common case for generating a reset segment is when a connection request
arrives and no process is listening on the destination port. We saw this
previously when we encountered the “connection refused” error messages. These
are common with TCP. In the case of UDP, an ICMP Destination Unreachable (Port
Unreachable) message is generated when a datagram arrives for a destination
port that is not in use. TCP uses a reset segment instead.

#### Aborting a Connection

The normal way to terminate a connection is for one side to send a FIN. This is
sometimes called an *orderly release*. It is also possible to abort a
connection by sending a reset instead of a FIN at any time. This is sometimes
called an *abortive release*.

#### Half-Open Connections

A TCP connection is said to be half-open if one end has closed or aborted the
con- nection without the knowledge of the other end. This can happen anytime
one of the peers crashes.

#### TIME-WAIT Assassination (TWA)

The TIME_WAIT state is intended to allow any datagrams lingering from a closed
connection to be discarded.  If, however, it receives certain segments from the
connection during this period, or more specifically an RST segment, it can
become desynchronized. This is called TIME-WAIT Assassination (TWA) [RFC1337].

### TCP Server Operation

#### TCP Port Numbers

TCP demultiplexes incoming segments using all four values that constitute the
local and foreign endpoints: destination IP address, destination port number,
source IP address, and source port number.

#### Restricting Local IP Addresses

The server application never sees the connection request. The rejection is done
by the operating system’s TCP module, based on the local address specified by
the application and the destination address contained in the arriving SYN seg-
ment. 

#### Restricting Foreign Endpoints

Address and port number binding options available to a TCP server

 Local Address  |    Foreign Address   |   Restricted to    | Comment
----------------|----------------------|--------------------|---------
 local_IP.lport | foraddr.foreign_port |     One client     | Not usually supported
 local_IP.lport |        \*.\*         | One local endpoint | Unusual (used by DNS servers)
 \*.local_port  |        \*.\*         |    One local port  | Most common; multiple address families (IPv4/IPv6) may be supported

#### Incoming Connection Queue

New connections may be in one of two distinct states before they are made
available to an applica- tion. The first case is connections that have not yet
completed but for which a SYN has been received (these are in the SYN_RCVD
state). The second case is connections that have already completed the
three-way handshake and are in the ESTABLISHED state but have not yet been
accepted by the application. Internally, the operating system ordinarily has
two distinct connection queues, one for each of these cases.

Traditionally, using the Berkeley sockets API, an application had only indirect
control of the sum of the sizes of these two queues. In modern Linux kernels
this behavior has been changed to be the number of connections in the second
case (ESTABLISHED connections).

In Linux, then, the following rules apply:

* When a connection request arrives (i.e., the SYN segment), the system-wide
  parameter net.ipv4.tcp_max_syn_backlog is checked (default 1000).
* Each listening endpoint has a fixed-length queue of connections that have
  been completely accepted by TCP (i.e., the three-way handshake is complete)
  but not yet accepted by the application. The application specifies a limit to
  this queue, commonly called the backlog. This backlog must be between 0 and a
  system-specific maximum called net.core.somaxconn, inclusive (default 128).
* If there is room on this listening endpoint’s queue for this new connection,
  the TCP module ACKs the SYN and completes the connection. The server
  application with the listening endpoint does not see this new connection
  until the third segment of the three-way handshake is received.
* If there is not enough room on the queue for the new connection, the TCP
  delays responding to the SYN, to give the application a chance to catch up.
  Linux is somewhat unique in this behavior—it persists in not ignoring
  incoming connections if it possibly can.

#### Attacks Involving TCP Connection Management

A SYN flood is a TCP DoS attack whereby one or more malicious clients generate
a series of TCP connection attempts (SYN segments) and send them at a server,
often with a “spoofed” (e.g., random) source IP address.

One mechanism invented to deal with this issue is called SYN cookies
[RFC4987]. The main insight with SYN cookies is that most of the information
that would be stored for a connection when a SYN arrives could be encoded
inside the Sequence Number field supplied with the SYN + ACK.

Another type of degradation attack on TCP involves PMTUD. In this case, an
attacker fabricates an ICMP PTB message containing a very small MTU value
(e.g., 68 bytes). This forces the victim TCP to attempt to fit its data into
very small packets, greatly reducing its performance.

Another type of attack involves disrupting an existing TCP connection and
possibly taking it over (called hijacking). These forms of attacks usually
involve a first step of “desynchronizing” the two TCP endpoints so that if they
were to talk to each other, they would be using invalid sequence numbers. They
are particular examples of sequence number attacks [RFC1948].

A collection of attacks generally called spoofing attacks involve TCP segments
that have been specially tailored by an attacker to disrupt or alter the
behavior of an existing TCP connection.


## TCP Timeout and Retransmission

TCP has two separate mechanisms for accomplishing retransmission, one based on
time and one based on the structure of the acknowledgments. The second approach
is usually much more efficient than the first.

TCP sets a timer when it sends data, and if the data is not acknowledged when
the timer expires, a timeout or timer-based retransmission of data occurs. The
timeout occurs after an interval called the *retransmission timeout* (RTO).

It has another way of initiating a retransmission called fast retransmission or
fast retransmit, which usually happens without any delay. Fast retransmit is
based on inferring losses by noticing when TCP’s cumulative acknowledgment
fails to advance in the ACKs received over time, or when ACKs carrying
selective acknowledgment information (SACKs) indicate that out-of-order
segments are present at the receiver.

### Simple Timeout and Retransmission Example

This doubling of time between successive retransmissions is called a *binary
exponential backoff*.

Logically, TCP has two thresholds to determine how persistently it will attempt
to resend the same segment.  Threshold R1 indicates the number of tries TCP
will make (or the amount of time it will wait) to resend a segment before
passing “negative advice” to the IP layer (e.g., causing it to reevaluate the
IP route it is using). Threshold R2 (larger than R1) dictates the point at
which TCP should abandon the connection.

### Setting the Retransmission Timeout (RTO)

Because TCP sends acknowledgments when it receives data, it is possible to send
a byte with a particular sequence number and measure the time required to
receive an acknowledgment that covers that sequence number. Each such mea-
surement is called an RTT sample. The challenge for TCP is to establish a good
estimate for the range of RTT values given a set of samples that vary over
time.

Say a packet is transmitted, a timeout occurs, the packet is retransmitted, and
an acknowledgment is received for it. Is the ACK for the first transmission or
the second? This is an example of the retransmission ambiguity problem.

TCP applies a backoff factor to the RTO, which doubles each time a subsequent
retransmission timer expires.

### Timer-Based Retransmission

TCP considers a timer-based retransmission as a fairly major event; it reacts
very cautiously when it happens by quickly reducing the rate at which it sends
data into the network. It does this in two ways. The first way is to reduce its
sending window size based on congestion control procedures. The other way is to
keep increasing a multiplicative backoff factor applied to the RTO each time a
retransmitted segment is again retransmitted.

### Fast Retransmit

Fast retransmit [RFC5681] is a TCP procedure that can induce a packet
retransmission based on feedback from the receiver instead of requiring a
retransmission timer to expire.

It is important to realize that TCP is required to generate an immediate
acknowledgment (a “duplicate ACK”) when an out-of-order segment is received,
and that the loss of a segment implies out-of- order arrivals at the receiver
when subsequent data arrives. When this happens, a hole is created at the
receiver. The sender’s job then becomes filling the receiver’s holes as quickly
and efficiently as possible.

### Retransmission with Selective Acknowledgments

When the SACK option is being used, an ACK can be augmented with up to three or
four SACK blocks that contain information about out-of-sequence data at the
receiver.

A SACK option that specifies n blocks has a length of 8n + 2 bytes, so the 40
bytes available to hold TCP options can specify a maximum of four blocks. It is
expected that SACK will often be used in conjunction with the TSOPT, which
takes an additional 10 bytes (plus 2 bytes of padding), meaning that SACK is
typically able to include only three blocks per ACK.

### Spurious Timeouts and Retransmissions


Under a number of circumstances, TCP may initiate a retransmission even when no
data has been lost. Such undesirable retransmissions are called spurious
retransmissions and are caused by spurious timeouts (timeouts firing too early)
and other reasons such as packet reordering, packet duplication, or lost
ACKs.

#### Duplicate SACK (DSACK) Extension

DSACK or D-SACK, which stands for duplicate SACK [RFC2883], is a rule, applied
at the SACK receiver and interoperable with conventional SACK senders, that
causes the first SACK block to indicate the sequence numbers of a duplicate
segment that has arrived at the receiver. The main purpose of DSACK is to
determine when a retransmission was not necessary and to learn additional facts
about the network.

#### The Eifel Detection Algorithm

The experimental Eifel Detection Algorithm [RFC3522] deals with this problem
using the TCP TSOPT to detect spurious retransmissions. After a retransmission
timeout occurs, Eifel awaits the next acceptable ACK. If the next acceptable
ACK indicates that the first copy of a retransmitted packet (called the
original transmit) was the cause for the ACK, the retransmission is considered
to be spurious.

#### Forward-RTO Recovery (F-RTO)

Forward-RTO Recovery (F-RTO) [RFC5682] is a standard algorithm for detecting
spurious retransmissions. It does not require any TCP options, so when it is
implemented in a sender, it can be used effectively even with an older
receiver that does not support the TCP TSOPT. It attempts to detect only
spurious retransmissions caused by expiration of the retransmission timer; it
does not deal with the other causes for spurious retransmissions or
duplications mentioned before.

#### The Eifel Response Algorithm

The Eifel Response Algorithm [RFC4015] is a standard set of operations to be
executed by a TCP once a retransmission has been deemed spurious. Because the
response algorithm is logically decoupled from the Eifel Detection Algorithm,
it can be used with any of the detection algorithms we just discussed. The
Eifel Response Algorithm was originally intended to operate for both
timer-based and fast retransmit spurious retransmissions but is currently
specified only for timerbased retransmissions.

### Packet Reordering and Duplication

#### Reordering


Packet reordering can occur in an IP network because IP provides no guarantee
that relative ordering between packets is maintained during delivery. This can
be beneficial (to IP at least), because IP can choose another path for traffic
(e.g., that is faster) without having to worry about the consequences that
doing so may cause traffic freshly injected into the network to pass ahead of
older traffic, resulting in the order of packet arrivals at the receiver not
matching the order of transmission at the sender. There are other reasons
packet reordering may occur.

The reordering of data segments has a somewhat different effect on TCP as
does reordering of ACK packets.  Because of asymmetric routing, it is
frequently the case that ACKs travel along different network links (and through
different routers) from data packets on
the forward path.

If reordering takes place in the reverse (ACK) direction, it causes the sending
TCP to receive some ACKs that move the window significantly forward followed by
some evidently old redundant ACKs that are discarded. This can lead to an
unwanted burstiness (instantaneous high-speed sending) behavior in the sending
pattern of TCP and also trouble in taking advantage of available network
bandwidth, because of the behavior of TCP’s congestion control

If fast retransmit were to be invoked whenever any duplicate ACK is received at
the sender, a large number of unnecessary retransmissions would occur on
network paths where a small amount of reordering is common. To handle this
situation, fast retransmit is triggered only after the duplicate threshold
(dupthresh) has been reached.

#### Duplication

Although rare, the IP protocol may deliver a single packet more than one time.
This can happen, for example, when a link-layer network protocol performs a
retransmission and creates two copies of the same packet. When duplicates are
created, TCP can become confused in some of the ways we have seen already.

#### Destination Metrics

TCP “learns” the characteristics of the network path between the sender and the
receiver over time. The learning is kept in state variables at the sender such
as srtt and rttvar. Some TCP implementations also keep track of an estimate of
the amount of packet reordering that has occurred recently along a path.rns”
the characteristics of the network path between the sender and the receiver
over time. The learning is kept in state variables at the sender such as srtt
and rttvar. Some TCP implementations also keep track of an estimate of the
amount of packet reordering that has occurred recently along a path.

When a new connection is created, TCP consults the data structure to see if
there is any preexisting information regarding the path to the destination host
with which it will be communicating. If so, initial values for srtt, rttvar,
and so on can be initialized to some value based on previous, relatively
recent experience.

#### Repacketization

When TCP times out and retransmits, it does not have to retransmit the identi-
cal segment. Instead, TCP is allowed to perform repacketization, sending a
bigger segment, which can increase performance.

#### Attacks Involving TCP Retransmission

There is a class of DoS attack called low-rate DoS attacks [KK03]. In such an
attack, an attacker sends bursts of traffic to a gateway or host, causing the
victim system to experience a retransmission timeout. Given an ability to
predict when the victim TCP will attempt to retransmit, the attacker generates
a burst of traffic at each retransmission attempt. As a consequence, the victim
TCP perceives congestion in the network, throttles its sending rate to near
zero, keeps backing off its RTO according to Karn’s algorithm, and effectively
receives very little network throughput.


## TCP Data Flow and Window Management

### Interactive Communication

Studies of TCP traffic usually find that 90% or more of all TCP segments
contain bulk data (e.g., Web, file sharing, electronic mail, backups) and the
remaining portion contains interactive data (e.g., remote login, network
games).

Each interactive keystroke normally generates a separate data packet.
Furthermore, *ssh* invokes a shell (command interpreter) on the remote system
(the server), which echoes the characters that are typed at the client.

A single typed character could thus generate four TCP segments: the inter-
active keystroke from the client, an acknowledgment of the keystroke from the
server, the echo of the keystroke from the server, and an acknowledgment of the
echo from the client back to the server.

### Delayed Acknowledgments

Using a cumulative ACK allows TCP to intentionally delay sending an ACK for
some amount of time, in the hope that it can combine the ACK it needs to send
with some data the local application wishes to send in the other direction.
This is a form of piggybacking that is used most often in conjunction with
bulk data transfers.

Delaying ACKs causes less traffic to be carried over the network than when ACKs
are not delayed because fewer ACKs are used. A ratio of 2 to 1 is fairly com-
mon for bulk transfers.

### Nagle Algorithm

When using IPv4, sending one single key press generates TCP/IPv4 packets of
about 88 bytes in size (using the encryption and authentication from the
example): 20 bytes for the IP header, 20 bytes for the TCP header (assuming no
options), and 48 bytes of data. These small packets (called tinygrams) have a
relatively high overhead for the network. A simple and elegant solution was
proposed by John Nagle in [RFC0896], now called the *Nagle algorithm*.

The Nagle algorithm says that when a TCP connection has outstanding data that
has not yet been acknowledged, small segments (those smaller than the SMSS)
cannot be sent until all outstanding data is acknowledged. Instead, small
amounts of data are collected by TCP and sent in a single segment when an
acknowledg- ment arrives. This procedure effectively forces TCP into
stop-and-wait behavior—it stops sending until an ACK is received for any
outstanding data. The beauty of this algorithm is that it is self-clocking: the
faster the ACKs come back, the faster the data is sent. On a comparatively
high-delay WAN, where reducing the number of tinygrams is desirable, fewer
segments are sent per unit time. Said another way, the RTT controls the packet
sending rate.

The trade-off the Nagle algorithm makes: fewer and larger packets are used, but
the required delay is higher.

#### Delayed ACK and Nagle Algorithm Interaction

The combination of delayed ACKs and the Nagle algorithm leads to a form of
deadlock (each side waiting for the other). Fortunately, this deadlock is not
permanent and is broken when the delayed ACK timer fires, which forces the
client to provide an ACK even if the client has no additional data to send.
However, the entire data transfer becomes idle during this deadlock period,
which is usually not desirable. The Nagle algorithm can be disabled in such
circumstances, as we saw with ssh.

### Flow Control and Window Management

When TCP-based applications are not busy doing other things, they are typically
able to consume any and all data TCP has received and queued for them, leading
to no change of the Window Size field as the connection progresses.

On slow systems, or when the application has other things to accomplish, data
may have arrived for the application, been acknowledged by TCP, and be sitting
in a queue waiting for the application to read or “consume” it. When TCP starts
to queue data in this way, the amount of space available to hold new incoming
data decreases, and this change is reflected by a decreasing value of the
Window Size field. Eventually, if the application does not read or otherwise
consume the data at all, TCP must take some action to cause the sender to cease
sending new data entirely, because there would be no place to put it on
arrival. This is accomplished by sending a window advertisement of zero (no
space).

#### Sliding Windows

Each endpoint of a TCP connection is capable of sending and receiving data. The
amount of data sent or received on a connection is maintained by a set of
window structures. For each active connection, each TCP endpoint maintains a
send window structure and a receive window structure.

Three terms are used to describe the movement of the right and left edges of
the window:

* The window closes as the left edge advances to the right. This happens when
  data that has been sent is acknowledged and the window size gets smaller.
* The window opens when the right edge moves to the right, allowing more data
  to be sent. This happens when the receiving process on the other end reads
  acknowledged data, freeing up space in its TCP receive buffer.
* The window shrinks when the right edge moves to the left. The Host
  Requirements RFC [RFC1122] strongly discourages this, but TCP must be able to
  cope with it.

If the left edge reaches the right edge, it is called a zero window. This stops
the sender from transmitting any data. If this happens, the sending TCP begins
to probe the peer’s window to look for an increase in the offered window.

For the receiver, any bytes received with sequence numbers less than the left
window edge (called RCV.NXT) are discarded as duplicates, and any bytes
received with sequence numbers beyond the right window edge (RCV.WND bytes
beyond RCV.NXT) are discarded as out of scope.

Note that the ACK number generated at the receiver may be advanced only when
segments fill in directly at the left window edge because of TCP’s cumulative
ACK structure. With selective ACKs, other in-window segments can be
acknowledged using the TCP SACK option, but ultimately the ACK number itself is
advanced only when data contiguous to the left window edge is received

#### Zero Windows and the TCP Persist Timer

We have seen that TCP implements flow control by having the receiver specify
the amount of data it is willing to accept from the sender: the receiver’s
advertised window. When the receiver’s advertised window goes to zero, the
sender is effectively stopped from transmitting data until the window becomes
nonzero. When the receiver once again has space available, it provides a window
update to the sender to indicate that data is permitted to flow once again.

If an acknowledgment (containing a window update) is lost, we could end up with
both sides waiting for the other. To prevent this form of deadlock from
occurring, the sender uses a persist timer to query the receiver periodically,
to find out if the window size has increased. The persist timer triggers the
transmission of window probes. Window probes are segments that force the
receiver to provide an ACK, which also necessarily contains a Win- dow Size
field.

#### Silly Window Syndrome (SWS)

Window-based flow control schemes, especially those that do not use fixed-size
segments (such as TCP), can fall victim to a condition known as the silly
window syndrome (SWS). When it occurs, small data segments are exchanged across
the connection instead of full-size segments [RFC0813]. This leads to
undesirable inef- ficiency because each segment has relatively high overhead—
a small number of data bytes relative to the number of bytes in the headers.

> See the example in this chapter which explains how TCP avoids SWS in details.

#### Large Buffers and Auto-Tuning

In this chapter, we have seen that an application using a small receive buffer
size may be doomed to significant throughput degradation compared to other
applica- tions using TCP in similar conditions. Even if the receiver specifies
a large enough buffer, the sender might specify too small a buffer, ultimately
leading to bad performance. This problem became so important that many TCP
stacks now decouple the allocation of the receive buffer from the size
specified by the application.

With auto-tuning, the amount of data that can be outstanding in the connection
is continuously estimated, and the advertised window is arranged to always be
at least this large (provided enough buffer space remains to do so). This has
the advantage of allowing TCP to achieve its maximum available throughput rate
(subject to the available network capacity) without having to allocate
excessively large buffers at the sender or receiver ahead of time.

### Urgent Mechanism

When the sender’s TCP receives such a write request, it enters a special state
called urgent mode. Upon entering urgent mode, it records the last byte the
application specified as urgent data. This is used to set the Urgent Pointer
field in each subsequent TCP header the sender generates until the application
ceases writing urgent data and all the sequence numbers up to the urgent
pointer have been acknowledged by the receiver.

A receiving TCP enters urgent mode when it receives a segment with the URG bit
field set. The receiving application can discover whether its TCP has entered
urgent mode using a standard socket API call (select()).

The operation of the urgent mechanism has been a source of confusion because
the Berkeley sockets API and documentation use the term *out-of-band* (OOB)
data, although in reality TCP does not implement any true OOB capability.

The “exit point” for urgent mode is defined to be the sum of the Sequence
Number field and the Urgent Pointer field in a TCP segment. Only one urgent
“point” (a sequence number offset) is maintained per TCP connection, so a
packet arriving with a valid Urgent Pointer field causes the information
contained in any previous urgent pointer to be lost.

### Attacks Involving Window Management

The window management procedures for TCP have been the subject of various
attacks, primarily forms of resource exhaustion. In essence, advertising a
small window slows a TCP transfer, tying up resources such as memory for a
potentially long time.


## TCP Congestion Control

When a router is forced to discard data because it cannot handle the arriving
traffic rate, is called congestion.

### Detection of Congestion in TCP

There is no explicit signaling about congestion. Instead, if a typical TCP is
to react somehow to congestion, it must first conclude that congestion is
occurring.

In TCP, an assumption is made that a lost packet is an indicator of congestion,
and that some response (i.e., slowing down in some way) is required.

When packets are detected as lost, it is TCP’s responsibility to resend them.
We are now concerned with what else TCP does when it observes a lost packet. In
particular, we are interested in how it interprets this as a signal that
congestion has occurred, and that it should slow down.

#### Slowing Down a TCP Sender

The new value used to hold the estimate of the network’s available capacity is
called the *congestion window*, written more compactly as simply *cwnd*.

    W = min(cwnd, awnd)

The total amount of data a sender has introduced into the network for which it
has not yet received an acknowledgment is sometimes called the *flight size*,
which is always less than or equal to W.

### The Classic Algorithms

TCP learns the value for awnd with one packet exchange to the receiver, but
without any explicit signaling, the only obvious way it has to learn a good
value for cwnd is to try sending data at faster and faster rates until it
experiences a packet drop (or other congestion indicator). This could be
accomplished by either sending immediately at the maximum rate it can (subject
to the value of awnd), or it could start more slowly.

Because of the detrimental effects on the performance of other TCP connections
sharing the same network path that could be experienced when starting at full
rate, a TCP generally uses one algorithm to avoid starting so fast when it
starts up to get to steady state. It uses a different one once it is in steady
state.

The arrival of an ACK, called the *ACK clock*, triggers the system to take the
action of sending another packet.  This relationship is sometimes called
*self-clocking*.

#### Slow Start

The slow start algorithm is executed when a new TCP connection is created or
when a loss has been detected due to a retransmission timeout (RTO). It may
also be invoked after a sending TCP has gone idle for some time. The purpose of
slow start is to help TCP find a value for cwnd before probing for more
available bandwidth using congestion avoidance and to establish the ACK clock.
Typically, a TCP begins a new connection in slow start, eventually drops a
packet, and then settles into steady-state operation using the congestion
avoidance algorithm.

A TCP begins in slow start by sending a certain number of segments (after the
SYN exchange), called the *initial window* (IW). The value of IW was originally
one SMSS, although with [RFC5681] it is allowed to be larger. The formula works
as follows:

    IW = 2*(SMSS) and not more than 2 segments (if SMSS > 2190 bytes)
    IW = 3*(SMSS) and not more than 3 segments (if 2190 ≥ SMSS > 1095 bytes)
    IW = 4*(SMSS) and not more than 4 segments (otherwise)

Slow start operates by incrementing cwnd by min(N, SMSS) for each good ACK
received, where N is the number of previously unacknowledged bytes ACKed by the
received “good ACK.”

The switch point at which TCP switches from operating in slow start to
operating in congestion avoidance is determined by the relationship between
cwnd and a value called the slow start threshold (or ssthresh).

#### Congestion Avoidance

Once ssthresh is established and cwnd is at least at this level, a TCP runs the
congestion avoidance algorithm, which seeks additional capacity by increasing
cwnd by approximately one segment for each window’s worth of data that is moved
from sender to receiver successfully.

#### Selecting between Slow Start and Congestion Avoidance

We mentioned ssthresh earlier. This threshold is a limit on the value of cwnd
that determines which algorithm is in operation, slow start or congestion
avoidance. When cwnd < ssthresh, slow start is used, and when cwnd > ssthresh,
congestion avoidance is used.

The initial value of ssthresh may be set arbitrarily high (e.g., to awnd or
higher), which causes TCP to always start with slow start. When a
retransmission occurs, caused by either a retransmission timeout or the
execution of fast retransmit, ssthresh is updated as follows:

    ssthresh = max(flight size/2, 2*SMSS)

#### Tahoe, Reno, and Fast Recovery

Fast recovery allows cwnd to (temporarily) grow by 1 SMSS for each ACK received
while recovering. The congestion window is therefore inflated for a period of
time, allowing an additional new packet to be sent for each ACK received,
until a good ACK is seen.

#### Standard TCP

To summarize the combined algorithm from [RFC5681], TCP begins a con- nection
in slow start (cwnd = IW) with a large value of ssthresh, generally at least
the value of awnd. Upon receiving a good ACK (one that acknowledges new data),
TCP updates the value of cwnd as follows:

    cwnd += SMSS (if cwnd < ssthresh)           Slow start
    cwnd += SMSS*SMSS/cwnd (if cwnd > ssthresh) Congestion avoidance

When fast retransmit is invoked because of receipt of a third duplicate ACK (or
other signal, if conventional fast retransmit initiation is not used), the
following actions are performed:

    1. ssthresh is updated to no more than the value given in equation above.
    2. The fast retransmit algorithm is performed, and cwnd is set to
       (ssthresh+3*SMSS).
    3. cwnd is temporarily increased by SMSS for each duplicate ACK received.
    4. When a good ACK is received, cwnd is reset back to ssthresh.

The actions in steps 2 and 3 constitute fast recovery.

Slow start is always used in two cases: when a new connection is started, and
when a retransmission timeout occurs. It can also be invoked when a sender has
been idle for a relatively long time or there is some other reason to suspect
that cwnd may not accurately reflect the current network congestion state.

### Evolution of the Standard Algorithms

> refs:
>
>   [1] Simulation-based Comparisons of Tahoe, Reno and SACK TCP
>       (http://www.eecs.berkeley.edu/~fox/summaries/networks/tcp_comp.html)

#### NewReno

NewReno is a popular variant of modern TCPs—it does not suffer from the
problems of the original fast recovery and is significantly less complicated to
implement than SACKs. With SACKs, however, a TCP can perform better than
NewReno when multiple packets are lost in a window of data, but doing this
requires careful attention to the congestion control procedures.

#### TCP Congestion Control with SACK

The following issue arises with SACK TCP: using only cwnd as a bound on the
sender’s sliding window to indicate how many (and which) packets to send during
recovery periods is not sufficient. Instead, the selection of which packets to
send needs to be decoupled from the choice of when to send them. Said another
way, SACK TCP underscores the need to separate the congestion management from
the selection and mechanism of packet retransmission.

One way to implement this decoupling is to have a TCP keep track of how much
data it has injected into the network separately from the maintenance of the
window. In [RFC3517] this is called the pipe variable, an estimate of the
flight size.

#### Forward Acknowledgment (FACK) and Rate Halving

In an effort to avoid the initial pause after loss but not violate the
convention of emerging from recovery with a congestion window set to half of
its size on entry, forward acknowledgment (FACK) was described in [MM96]. It
consists of two algorithms called “overdamping” and “rampdown.”

The basic operation of *Rate-Halving with Bounding Parameters* (RHBP) allows the
TCP sender to send one packet for every two duplicate ACKs it receives during
one RTT.

To keep an accurate estimate of the flight size, RHBP uses information from
SACKs to determine the FACK: the highest sequence number known to have reached
the receiver, plus 1.

The following expression allows the RHBP sender to transmit, if satisfied:

    (SND.NXT – fack + retran_data + len) < cwnd

#### Limited Transmit

With limited transmit, a TCP with unsent data is permitted to send a new packet
for each pair of consecutive duplicate ACKs it receives. Doing this helps to
keep at least a minimal number of packets in the network—enough so that fast
retransmit can be triggered upon packet loss. 

#### Congestion Window Validation (CWV)

The CWV algorithm work as follows: Whenever a new packet is to be sent, the
time since the last send event is measured to determine if it exceeds one RTO.
If so,

    * ssthresh is modified but not reduced—it is set to max(ssthresh,
      (3/4)*cwnd).
    * cwnd is reduced by half for each RTT of idle time but is always at least
      1 SMSS.

For application-limited periods that are not idle, the following similar
behavior is used:

    * The amount of window actually used is stored in W_used.
    * ssthresh is modified but not reduced—it is set to max(ssthresh,
      (3/4)*cwnd).
    * cwnd is set to the average of cwnd and W_used.

### Handling Spurious RTOs—the Eifel Response Algorithm

The Eifel Response Algorithm is aimed at handling the retransmission timer and
congestion control state after a retransmission timer has expired. Here we
discuss only the congestion-related portions of the response algorithm. It is
initi- ated after the first timeout-based retransmission is sent. Its purpose
is to undo a change to ssthresh when a retransmission is deemed to be spurious.

    pipe_prev = min(flight size, ssthresh)

If the RTO is spurious, the following steps are executed when an ACK arrives
after the retransmission:

    1. If a received good ACK includes an ECN-Echo flag, stop (see Section
       16.11).
    2. cwnd = flight size + min(bytes_acked, IW) (assuming cwnd is measured in
       bytes).

### Sharing Congestion State

In an effort to generalize this idea and extend it to protocols and
applications other than TCP, [RFC3124] describes the Congestion Manager, which
provides a local operating system service available to protocol implementations
to learn information such as path loss rate, estimated congestion, RTT, and so
forth to destination hosts.

### TCP Friendliness

To provide a guideline for protocol designers to avoid unfairly competing with
TCP flows when operating cooperatively on the Internet, researchers have
developed an equation-based rate control limit that provides a bound of the
bandwidth used by a conventional TCP connection operating in a particular
environment. This method is called TCP Friendly Rate Control (TFRC)
[RFC5348][FHPW00]. It is designed to provide a sending rate limit based on a
combination of connection parameters and with environmental factors such as RTT
and packet drop rate.

### TCP in High-Speed Environments

#### HighSpeed TCP (HSTCP) and Limited Slow Start

With *limited slow start*, a new parameter called max_ssthresh is introduced.
This value is not the maximum value of ssthresh but instead a threshold for
cwnd that works as follows: If cwnd <= max_ssthresh, slow start proceeds as
normal. If max_ssthresh < cwnd <= ssthresh, then cwnd is increased by at most
(max_ssthresh/2) SMSS per RTT.

    if (cwnd <= max_ssthresh) {
        cwnd = cwnd + SMSS /*regular slow start*/
    } else {
        K = int(cwnd / (0.5 * max_ssthresh))
        cwnd = cwnd + int((1/K)*SMSS) /*limited slow start*/
    }

#### Binary Increase Congestion Control (BIC and CUBIC)

##### BIC-TCP

The main goal of BIC TCP is to provide linear RTT fairness even though
congestion windows may be quite large (which is required to use high-bandwidth
links). Linear RTT fairness means that connections receive a bandwidth share
inversely proportional to their RTTs, rather than some more complicated or
unknown function.

The approach modifies a standard TCP sender with two algorithms: binary search
increase and additive increase. These algorithms are invoked after a congestion
indication (e.g., packet loss), but only one of the algorithms is in operation
at any given point in time.

The binary search increase algorithm operates as follows: The current minimum
window is the last point at which the connection experienced no packet loss
during an entire RTT. The maximum window is the window size at which the
connection last experienced loss, if known. The desired window lies somewhere
between the two. Using a binary search technique, BIC-TCP selects a trial
window in the midpoint of these two values and tries again recursively. If this
point shows continued packet loss, it becomes the new maximum and the process
repeats. If not, it becomes the new minimum and the process repeats.

##### CUBIC

It addresses concerns raised that BIC-TCP may be too aggressive under some
circumstances. It also simplifies the window growth procedures. Instead of
using a threshold (Smax) to decide when to invoke the binary search increase
versus additive increase, an odd-degree polynomial function, in particular a
cubic function, is used instead to control the window increase function. Cubic
functions can have both convex and concave portions, meaning that they can grow
more slowly in some portions (concave) and more quickly in others (convex).

### Delay-Based Congestion Control

#### Vegas

Vegas operates by estimating the amount of data it expects to transfer in a
certain amount of time and comparing this with the amount of data it is
actually able to transfer. If the requisite amount of data is not transferred,
it is likely to be held up in a router queue along the path. If this condition
persists, the Vegas sender slows down. This is in contrast to the standard TCP
approach, which forces a packet drop to occur in order to determine the point
at which the network is congested.

While in its congestion avoidance phase, during each RTT, Vegas measures the
amount of data transferred and divides this number by the minimum delay
observed across the connection. It maintains two thresholds, α and β (where α <
β). When the difference in expected throughput (window size divided by the
smallest RTT observed) versus achieved throughput is less than α, the
congestion window is increased; when it is greater than β, the congestion
window is decreased. Otherwise, it is left as is. All changes to the congestion
window are linear, meaning the scheme is an *additive increase/additive
decrease* (AIAD) congestion control scheme.

### Buffer Bloat

Perhaps ironically, this large amount of memory (as compared to traditional
networking devices) can actually lead to degraded performance for protocols
such as TCP. This problem has been termed *buffer bloat*.

The standard TCP congestion control algorithms, which tend to keep buffers full
at bottleneck links, do not operate well when a large amount of buffering
occurs between the sender and receiver because the congestion indicator (a
packet drop) takes a long time to be delivered to a sender.

### Active Queue Management and ECN

Routers that apply scheduling and buffer management policies other than
FIFO/drop tail are usually said to be active, and the corresponding methods
they use to manage their queues are called *active queue management* (AQM)
mechanisms.

Although AQM can be useful independently, it becomes more useful when routers
and switches implementing AQM have a common method for conveying their status
to the end systems. For TCP, this is described in [RFC3168] and extended with
additional security in an experimental specification [RFC3540]. These RFCs
describe *Explicit Congestion Notification* (ECN), which is a way for routers
to mark packets (by ensuring both of the ECN bits in the IP header are set) to
indicate the onset of congestion.

Because the receiver normally returns information to the sender by using
(unreliable) ACK packets, there is a significant chance that the congestion
indicator could be lost. For this reason, TCP implements a small
reliability-enhancing protocol for carrying the indication back to the sender.
Upon receiving an incoming packet with CE set, the TCP receiver sets the
ECN-Echo bit field in each ACK packet it sends until receiving a CWR bit field
set to 1 from the TCP sender in a subsequent data packet. The CWR bit field
being set indicates that the congestion window (i.e., sending rate) has been
reduced.

### Attacks Involving TCP Congestion Control

Perhaps the earliest attack involves the fabrication of ICMPv4 Source Quench
messages. When these are delivered to a host running TCP, any connection to the
IP address contained in the offending datagram inside the ICMP message slows
down.

A more sophisticated and more recent set of attacks have been considered by
looking at *misbehaving receivers*. The attacks are named *ACK division*,
*DupACK spoofing*, and *Optimistic ACKing* and are implemented in a TCP variant
the authors (jokingly) call “TCP Daytona.”

ACK division operates by producing more than one ACK for the range of bytes
being acknowledged. Because the TCP congestion control typically operates based
on the arrival of ACK packets (rather than the ACK field contained in the ACK
itself), a sending TCP can be induced to increase cwnd faster than it would
otherwise.

DupACK spoofing causes a sender to increase its congestion window during fast
recovery. Recall from the previous discussion that during standard fast
recovery, cwnd is incremented for each duplicate ACK received. The attack
involves creating extra duplicate ACKs that cause this to happen more quickly
than intended.

Optimistic ACKing involves producing ACKs for segments that have not yet
arrived. Because TCP’s congestion control computations are based on end-to-end
RTTs, ACKing data that has not yet arrived causes the sender to react faster
than it would because it is fooled into believing the actual RTT is smaller.


## TCP Keepalive

### Introduction

Under some circumstances, it is useful for a client or server to become aware
of the termination or loss of connection with its peer. In other circumstances,
it is desirable to keep a minimal amount of data flowing over a connection,
even if the applications do not have any to exchange. TCP *keepalive* provides
a capability useful for both cases. Keepalive is a method for TCP to probe its
peer without affecting the content of the data stream. It is driven by a
*keepalive timer*. When the timer fires, a keepalive *probe* (keepalive for
short) is sent, and the peer receiving the probe responds with an ACK.

> Keepalives are not part of the TCP specification.

### Description

Either end of a TCP connection may request keepalives, which are turned off by
default, for their respective direction of the connection. A keepalive can be
set for one side, both sides, or neither side.

If there is no activity on the connection for some period of time (called the
*keepalive time*), the side(s) with keepalive enabled sends a keepalive probe to
its peer(s). If no response is received, the probe is repeated periodically
with a period set by the *keepalive interval* until a number of probes equal to
the number *keepalive probes* is reached. If this happens, the peer’s system is
determined to be unreachable and the connection is terminated.

A keepalive probe is an empty (or 1-byte) segment with sequence number equal to
one less than the largest ACK number seen from the peer so far.

Anytime it is operating, a TCP using keepalives may find its peer in one of
four states:

* The peer host is still up and running and reachable.
* The peer’s host has crashed and is either down or in the process of
  rebooting.
* The client’s host has crashed and rebooted.
* The peer’s host is up and running but is unreachable from the requestor for
  some reason.

### Attacks Involving TCP Keepalives

TCP keepalives contain no user-level data, so the use of encryption is limited
at best. The consequence is that TCP keepalives may be spoofed. When TCP
keepalives are spoofed, the victim can be coerced into keeping resources
allocated for a period longer than intended.

A passive observer could notice the existence of keepalives and their
interarrival times to conceivably learn information about the configuration
parameters (possibly identifying the type of sending system, called
*fingerprinting*) or about the network topology (i.e., whether downstream
routers are forwarding traffic or not).

## Security: EAP, IPsec, TLS, DNSSEC, and DKIM

### Basic Principles of Information Security

* *Confidentiality* means that information is made known only to its intended
  users (which could include processing systems).
* *Integrity* means that information has not been modified in an unauthorized
  way before it is delivered.
* *Availability* means that information is available when needed.

### Threats to Network Communication

Attacks can generally be categorized as either passive or active.

Passive attacks are mounted by monitoring or eaves- dropping on the contents of
network traffic, and if not handled they can lead to unauthorized release of
information (loss of confidentiality).

Active attacks can cause modification of information (with possible loss of
integrity) or denial of service (loss of availability).

### Basic Cryptography and Security Mechanisms

#### Cryptosystems

Two most important types of cryptographic algorithms: *symmetric key* and
*public (asymmetric) key* ciphers.

A *cleartext* message is processed by an encryption algorithm to produce
ciphertext (scrambled text). The key is a particular sequence of bits used to
drive the *encryption algorithm* or cipher. With different keys, the same input
produces different outputs. Combining the algorithms with supporting protocols
and operating methods forms a *cryptosystem*.

One of the major benefits of asymmetric key cryptosystems is that secret key
material does not have to be securely distributed to every party that wishes to
communicate.

A symmetric encryption algorithm is usually classified as either a *block
cipher* or a *stream cipher*.

Block ciphers perform operations on a fixed number of bits (e.g., 64 or 128) at
a time, and stream ciphers operate continuously on however many bits (or bytes)
are provided as input.

In a *hybrid* cryptosystem, elements of both public key and symmetric key
cryptography are used. Most often, public key operations are used to exchange a
randomly generated confidential (symmetric) session key, which is used to
encrypt traffic for a single transaction using a symmetric algorithm. The
reason for doing so is performance—symmetric key operations are less
computationally intensive than public key operations.

#### Rivest, Shamir, and Adleman (RSA) Public Key Cryptography

We have seen how public key cryptography can be used for both digital
signatures and confidentiality. The most common approach is called RSA in
deference to its authors’ names, Rivest, Shamir, and Adleman. The security of
this system hinges on the difficulty of factoring large numbers into
constituent primes.

Key generation:

1. Choose two distinct prime numbers p and q
2. Compute n = pq
3. Compute φ(n) = φ(p)φ(q) = (p − 1)(q − 1)
4. Choose an integer e such that 1 < e < φ(n) and gcd(e, φ(n)) = 1
5. Determine d as de ≡ 1 (mod φ(n))

The public key is (n, e), the private key is (n, d).

Encryption:

1. Turn message M into integer m, 0 ≤ m < n
2. Compute the ciphertext c ≡ m^e (mod n)

Decryption:

1. Recover message m with m ≡ c^d (mod n)

Proofs of correctness:

Use *Fermat's little theorem*.

> http://en.wikipedia.org/wiki/RSA_(cryptosystem)#Proofs_of_correctness

Security:

The security of RSA is based on the difficulty of factoring large numbers.

#### Diffie-Hellman-Merkle Key Agreement (aka Diffie-Hellman or DH)

Doing so in a network that may contain eavesdroppers (such as Eve) is a
challenge, because it is not immediately obvious how to have two principals
(such as Alice and Bob) agree on a common secret number without Eve knowing.
The *Diffie-Hellman-Merkle Key Agreement protocol* (more commonly called simply
Diffie-Hellman or DH) provides a method for accomplishing this task, based on
the use of finite field arithmetic.

Key generalization:

1. Alice and Bob agree on a finite cyclic group G and a generating element g in
   G. (This is usually done long before the rest of the protocol; g is assumed
   to be known by all attackers.) We will write the group G multiplicatively.
2. Alice picks a random natural number a and sends *g^a mod p* to Bob.
3. Bob picks a random natural number b and sends *g^b mod p* to Alice.
4. Alice computes *(g^b)^a mod p*.
5. Bob computes *(g^a)^b mod p*.

Both Alice and Bob are now in possession of the group element *g^ab mod p*,
which can serve as the shared secret key.

Security:

Let p be a (large) prime number and g < p be a primitive root mod p. With these
assumptions, every integer in the group Z\_p = {1, ..., p - 1} can be generated
by raising g to some power. Said another way, for every n, there exists some k
for which g^k ≡ n (mod p). Finding the value (or values) of k given g, n, and p
(called the *discrete logarithm problem*) is considered to be difficult,
resulting in the belief that DH is secure.

However, this basic protocol is vulnerable to an attack from Mallory. Mallory
can pretend to be Bob when communicating with Alice and vice versa by supplying
her own A and B values. However, the basic DH protocol can be extended to
protect from this man-in-the-middle attack if the public values for A and B
are authenticated.

#### Signcryption and Elliptic Curve Cryptography (ECC)

Reducing the effort of combining digital signatures and encryption for
confidentiality, a class of signcryption schemes (also called authenticated
encryption) provides both features at a cost less than the sum of the two if
computed separately.

An alternative based on the difficulty of finding the discrete logarithm of an
elliptic curve element has emerged, known as elliptic curve cryptography (ECC,
not to be confused with error-correcting code).

#### Key Derivation and Perfect Forward Secrecy (PFS)

The session key is ordinarily a random number (see the following section)
generated by a function called a *key derivation function (KDF)*, based on some
input such as a master key or a previous session key. A scheme in which the
compromise of one session key keeps future communications secure is said to
have *perfect forward secrecy (PFS)*.

#### Pseudorandom Numbers, Generators, and Function Families

Given that computers are not very random by nature, obtaining true random
numbers is somewhat difficult. The numbers used in most computers for
simulating randomness are called pseudorandom numbers.

Pseudorandom numbers are produced by an algorithm or device known as a
*pseudorandom number generator (PRNG)* or *pseudorandom generator (PRG)*,
depending on the author.

A *pseudorandom function family (PRF)* is a family of functions that appear to
be algorithmically indistinguishable (by polynomial time algorithms) from truly
random functions [GGM86]. A PRF is a stronger concept than a PRG, as a PRG can
be created from a PRF. PRFs are the basis for *cryptographically strong* (or
secure) pseudorandom number generators, called CSPRNGs.

#### Nonces and Salt

A *cryptographic nonce* is a number that is used once (or for one transaction)
in a cryptographic protocol. Most commonly, a nonce is a random or pseudorandom
number that is used in authentication protocols to ensure freshness. Freshness
is the (desirable) property that a message or operation has taken place in the
very recent past.

A *salt or salt value*, used in the cryptographic context, is a random or
pseudorandom number used to frustrate brute-force attacks on secrets.
Brute-force attacks usually involve repeatedly guessing a password, passphrase,
key, or equivalent secret value and checking to see if the guess was correct.
Salts work by frustrating the checking portion of a brute-force attack.

At the time, the encryption method (DES) was well known and there was concern
that a hardware-based dictionary attack would be possible whereby many words
from a dictionary were encrypted with DES ahead of time (forming a *rainbow
table*) and compared against the password file.

#### Cryptographic Hash Functions and Message Digests

A checksum or FCS can be used to verify message integrity against an adversary
like Mallory if properly constructed using special functions. Such functions
are called *cryptographic hash functions* and often resemble portions of
encryption algorithms. The output of a cryptographic hash function H, when
provided a message M, is called the *digest* or *fingerprint* of the message,
H(M).

A message digest is a type of strong FCS that is easy to compute and has the
following important properties:

* *Preimage resistance*: Given H(M), it should be difficult to determine M if not
  already known.
* *Second preimage resistance*: Given H(M1), it should be difficult to
  determine an M2 ≠ M1 such that H(M1) = H(M2).
* *Collision resistance*: It should be difficult to find any pair M1, M2 where
  H(M1) = H(M2) when M2 ≠ M1.

The two most common cryptographic hash algorithms are at present the *Message
Digest Algorithm 5* (MD5, [RFC1321]), which produces a 128-bit (16-byte)
digest, and the *Secure Hash Algorithm 1* (SHA-1), which produces a 160-bit
(20-byte) digest.

#### Message Authentication Codes (MACs, HMAC, CMAC, and GMAC)

A message authentication code (unfortunately abbreviated MAC or sometimes MIC
but unrelated to the link-layer MAC addresses we discussed in Chapter 3) can be
used to ensure message integrity and authentication.

MACs require resistance to various forms of forgery. For a given keyed hash
function H(M,K) taking input message M and key K, resistance to *selective
forgery* means that it is difficult for an adversary not knowing K to form
H(M,K) given a specific M. H(M,K) is resistant to *existential forgery* if it
is difficult for an adversary lacking K to find any previously unknown valid
combination of M and H(M,K).

A standard MAC that uses cryptographic hash functions in a particular way is
called the *keyed-hash message authentication code (HMAC)*.

    HMAC-H (K, M)^t = Λ_t (H((K ⊕ opad)||H((K ⊕ ipad)||M)))

In this definition, opad (outer pad) is an array containing the value 0x5C
repeated |K| times, and ipad (inner pad) is an array containing the value 0x36
repeated |K| times. ⊕ is the vector XOR operator, and || is the concatenation
oper- ator. Normally the HMAC output is intended to be a certain number t of
bytes in length, so the operator Λ\_t(M) takes the left-most t bytes of M.

#### Cryptographic Suites and Cipher Suites

The combination of techniques used in a particular system, especially those we
see used with Internet protocols, are called a *cryptographic suite* or
sometimes a *cipher suite*, although the first term is more accurate.

### Certificates, Certificate Authorities (CAs), and PKIs

One of the challenges with public key cryptosystems is to determine the correct
public key for a principal or identity.

One model, called a *web of trust*, involves having a certificate (identity/key
binding) endorsed by a collection of existing users (called endorsers). An
endorser signs a certificate and distributes the signed certificate. The more
endorsers for a certificate over time, the more reliable it is likely to be. An
entity checking a certificate might require some number of endorsers or
possibly some particular endorsers to trust the certificate. The web of trust
model is decentralized and “grassroots” in nature, with no central authority.

The web of trust model was first described as part of the *Pretty Good Privacy
(PGP)* encryption system for electronic mail, which has evolved to support a
standard encoding format called OpenPGP, defined by [RFC4880].

A more formal approach, which has the added benefit of being provably secure
under certain theoretical assumptions in exchange for more dependence on a
centralized authority, involves the use of a *public key infrastructure (PKI)*.
A PKI is a service responsible for creating, revoking, distributing, and
updating key pairs and certificates. It operates with a collection of
*certificate authorities (CAs)*.

#### Public Key Certificates, Certificate Authorities, and X.509

While several types of certificates have been used in the past, the one of most
interest to us is based on an Internet profile of the ITU-T X.509 standard
[RFC5280]. In addition, any particular certificate may be stored and exchanged
in a number of file or encoding formats. The most common ones include DER, PEM
(a Base64 encoded version of DER), PKCS#7 (P7B), and PKCS#12 (PFX).

#### Validating and Revoking Certificates

To validate a certificate, a validation or certification path must be
established that includes a set of validated certificates, usually up to some
trust anchor (e.g., root certificate) that is already known to the relying
party.

There are several reasons why a certificate may need to be revoked, such as
when a certificate’s subject (or issuer) changes affiliations or name. When a
certificate is revoked, it may no longer be used. The challenge is to ensure
that entities that wish to use a certificate become aware if it has been
revoked.  In the Internet, there are two primary ways this is accomplished:
*CRLs* and the *Online Certificate Status Protocol (OCSP)* [RFC2560].

### TCP/IP Security Protocols and Layering

 Layer Number | Layer Name  |                       Examples
--------------|-------------|---------------------------------------------------------
      7       | Application | DNSSEC, DKIM, Diameter, RADIUS, SSH, Kerobse, IPsec(IKE)
      4       |  Transport  |                    TLS, DTLS, PANA
      3       |   Network   |                      IPsec (ESP)
      2       |    Link     |    802.1X(EAPoL), 802.1AE(MACSec), 802.11i/WPA2, EAP

### Network Access Control: 802.1X, 802.1AE, EAP, and PANA

*Network Access Control (NAC)* refers to methods used to authorize or deny
network communications to particular systems or users. Defined by the IEEE, the
*802.1X Port-Based Network Access Control (PNAC)* standard is commonly used
with TCP/ IP networks to support LAN security in enterprises, for both wired
and wireless networks.

Used in conjunction with the IETF standard *Extensible Authentication Protocol
(EAP)* [RFC3748], 802.1X is sometimes called *EAP over LAN (EAPoL)*, although
the 802.1X standard covers more than just the EAPoL packet format.

EAP can be used with multiple link-layer technologies and supports multiple
methods for implementing *authentication, authorization, and accounting (AAA)*.
EAP does not perform encryption itself, so it must be used in conjunction with
some other cryptographically strong protocol to be secure.

### Layer 3 IP Security (IPsec)

IPsec is an architecture and collection of standards that provide data source
authentication, integrity, confidentiality, and access control at the network
layer for IPv4 and IPv6 [RFC4301], including Mobile IPv6 [RFC4877]. It also
provides a way to exchange cryptographic keys between two communicating
parties, a recommended set of cryptographic suites, and a method for signaling
the use of compression.

The operation of IPsec can be divided into the establishment phase, where key
material is exchanged and a *security association (SA)* is built, followed by
the data exchange phase, where different types of encapsulation schemes, called
the *Authentication Header (AH)* and *Encapsulating Security Payload (ESP)*,
may be used in different modes such as tunnel mode or transport mode to protect
the flow of IP datagrams.

#### Internet Key Exchange (IKEv2) Protocol

An SA is a simplex (one-direction) authenticated association established
between two communicating parties, or between a sender and multiple receivers
if IPsec is supporting multicast. Most frequently, communication is
bidirectional between two parties, so a pair of SAs is required to use IPsec
effectively. A special protocol called the *Internet Key Exchange*.

#### Authentication Header (AH)

Defined in [RFC4302], the IP *Authentication Header (AH)*, one of the three
major components of IPsec, is an optional portion of the IPsec protocol suite
that provides a method for achieving origin authentication and integrity (but
not confidentiality) of IP datagrams.

#### Encapsulating Security Payload (ESP)

The ESP protocol of IPsec, defined in [RFC4303] (where it is called ESP (v3)
even though ESP provides no formal version numbers), provides a selectable
combina- tion of confidentiality, integrity, origin authentication, and
anti-replay protection for IP datagrams.

Given its flexibility and feature set, ESP is (far) more popular than AH.

#### L2TP/IPsec

The Layer 2 Tunneling Protocol (L2TP) supports tunneling of layer 2 traffic
such as PPP through IP and non-IP networks. It relies on authentication methods
that provide some authentication during connection initiation, but no
subsequent per-packet authentication, integrity protection, or confidentiality.
To address this concern, L2TP can be combined with IPsec [RFC3193]. The
combination, called L2TP/IPsec, provides a recommended method to establish
remote layer 2 VPN access to enterprise (or home) networks.

L2TP can be secured with IPsec using either a direct L2TP-over-IP encapsulation
(protocol number 115) or a UDP/IP encapsulation that eases NAT traversal.

#### IPsec NAT Traversal

Using NATs with IPsec can present something of a challenge, primarily because
IP addresses have traditionally been used in identifying communication
endpoints and are assumed to not change.

The primary approach to dealing with most of the NAT traversal concerns is to
encapsulate IPsec ESP and IKE traffic using UDP/IP, which can be modified by
conventional NATs when necessary. (There is no supported solution for NAT
traversal of AH.)

### Transport Layer Security (TLS and DTLS)

The most widely used protocol for security operates just above the transport
layer and is called Transport Layer Security (TLS).

#### TLS 1.2

Confidentiality and data integrity are provided based on a variety of
cryptographic suites that use certificates that can be provided by a PKI. TLS
can also establish secure connections between two anonymous parties (without
using certificates), but this application is vulnerable to a MITM attack (not
surprising, given that each end is not even strongly identified).

The TLS protocol has two layers of its own, called the *record layer* and the
*upper layer*.

#### TLS with Datagrams (DTLS)

The TLS protocol assumes a stream-based underlying transport protocol for
delivering its messages. A *datagram version (DTLS)* relaxes this assumption
but aims to otherwise achieve the same security goals as TLS using essentially
all the same message formats.

### DNS Security (DNSSEC)

Security for DNS covers both data within the DNS (resource records or RRs) as
well as security of transactions that synchronize or update contents of DNS
servers.

The mechanisms are called the *Domain Name System Security Extensions (DNSSEC)*
and are discussed in a family of RFCs [RFC4033][RFC4034] [RFC4035].

DNSSEC accommodates resolvers with varying levels of security “awareness.” A
*validating security-aware resolver* (also called *validating resolver*) checks
cryptographic signatures to ensure that the DNS data it handles is secure.

When operating, they are able to ascertain whether DNS information is *secure*
(valid with all signatures checked), *insecure* (valid signatures indicate that
something should not be present but is), *bogus* (proper data appears to be
present but cannot be validated for some reason), or *indeterminate* (veracity
cannot be determined, usually because of lack of signatures). The indeterminate
case is the default case when no other information is available.

DNSSEC works securely only when a zone is signed by a domain administrator,
there is some basis for trust, and both server and resolver software
participate.

#### DNSSEC Resource Records

As specified in [RFC4034], DNSSEC uses four new resource records (RRs) and two
message header bits (CD and AD).

* DNS Security (DNSKEY) Resource Records: DNSSEC uses the DNSKEY resource
  record to hold public keys.
* Delegation Signer (DS) Resource Records: A delegation signer (DS) resource
  record is used to refer to a DNSKEY RR, usually from a parent zone to a
  descendant zone.
* NextSECure (NSEC and NSEC3) Resource Records: The NextSECure (NSEC) RR is
  used to hold the “next” RRset owner’s domain name in the canonical ordering
  of names or a delegation point NS type RRset.
* Resource Record Signature (RRSIG) Resource Records: DNSSEC signs and
  validates signatures on RRsets using the Resource Record Signature (RRSIG)
  RR, and every authoritative RR in a zone must be signed (glue records and
  delegation NS records present in parent zones aren’t).

> There is a detailed example of DNSSEC usage in chapter 18.10.2.2

#### Transaction Authentication (TSIG, TKEY, and SIG(0))

With transaction authentication, the exchange between a particular resolver
and server (or between servers) is protected. Note, however, that transactional
security does not directly protect the contents of the DNS, as does DNSSEC. As
a result, DNSSEC and transaction authentication are complementary and can be
deployed together.

There are two primary methods for authenticating DNS transactions: TSIG and
SIG(0). TSIG uses shared keys and SIG(0) uses public/private key pairs.

### DomainKeys Identified Mail (DKIM)

*DomainKeys Identified Mail (DKIM)* [RFC5585] is intended to provide an
association between an entity and a domain name that can be used to help
determine the party responsible for originating a message, especially in the
e-mail context.
