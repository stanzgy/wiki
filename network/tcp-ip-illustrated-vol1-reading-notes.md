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

#### Simultaneous Open and Close Transitions

When a simultaneous open occurs, both ends send a SYN at about the same time,
entering the SYN_SENT state. When each end receives its peer’s SYN segments,
the state changes to SYN_RCVD, and each end resends a SYN and acknowledges the
received SYN. When each end receives the SYN plus the ACK, the state changes to
ESTABLISHED.

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


