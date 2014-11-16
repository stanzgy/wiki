# tcpkill工作原理分析

日常工作生活中大家在维护自己的服务器、VPS有时会碰到这样的情况：服务器上突然出现了许多来自未知ip的网络连接与流量，我们需要第一时间切断这些可能有害的网络连接。除了iptables/ipset, blackhole routing这些常规手段，我们还可以借助一些更轻量级的小工具来临时处理这些情况，如tcpkill。


## tcpkill使用简介

tcpkill是一个网络分析工具集dsniff中的一个小工具。在Debian/Ubuntu上可以直接通过dsniff包安装：

	# aptitude install dsniff

tcpkill使用的语法和tcpdump几乎一致：

	tcpkill [-i interface] [-1...9] expression
	
其中第一个参数 `-i` 指定网卡设备。

第二个参数指定“kill”的强制等级，越高越强，默认为3，我们在后面了解tcpkill的工作原理后会知道这个参数的具体作用。

第三个参数则是匹配需要kill的tcp连接通配表达式，语法与tcpdump使用的pcap-filter完全一样。

如果我们需要使用tcpkill临时禁止服务器与主机10.0.0.1的tcp连接，可以在服务器上输入命令：

	# tcpkill host 10.0.0.1
	
tcpkill会一直阻止主机10.0.0.1与服务器的网络连接，直到你结束这个tcpkill进程位置。


## tcpkill原理分析

在使用tcpkill时，会发现一件奇怪的事情，运行tcpkill命令后并不会马上中断匹配的tcp连接，只有当该连接有新的tcp包发送接收时，tcpkill才会“kill”这个tcp连接。这个奇怪的现象燃起了我们的好奇心，于是探索一下tcpkill到底是如何工作的。

下面以Linux下的nc命令为例。

我们在两个主机hostA与hostB间通过nc命令建立一个tcp连接：

hostA在本地tcp 5555端口监听

	hostA$ nc -l -p 5555

hostB通过本地6666端口连接hostA的5555端口
	
	hostB$ nc hostA 5555 -p 6666
	
此时在hostA上已经可以观察到一条与hostB的ESTABLISHED连接

	hostA$ netstat -anp|grep 5555
	tcp        0      0 hostA:5555    hostB:6666     ESTABLISHED 19638/nc

在hostA上通过tcpdump也可以观察到3次握手已经完成

	hostA# tcpdump -i eth1 port 5555
	IP hostB.6666 > hostA.5555: Flags [S], seq 750827752, ...
	IP hostA.5555 > hostB.6666: Flags [S.], seq 1191909671, ack 750827753, ...
	IP hostB.6666 > hostA.5555: Flags [.], ack 1, win 115, ...

如果此时运行tcpkill命令尝试“kill”这个tcp连接

	hostA# tcpkill -1 -i eth1 port 5555
	tcpkill: listening on eth1 [port 5555]

会发现hostA与hostB上的nc命令并没有受到任何影响而退出，hostA上观察到该tcp连接还是ESTABLISHED状态，tcpdump与tcpkill也没有任何新的输出。

	hostA$ netstat -anp|grep 5555
	tcp        0      0 hostA:5555    hostB:6666     ESTABLISHED 19638/nc

运行tcpkill命令后，建立好的tcp连接并没有受到任何影响。

如果我们此时在hostB的nc上输入任意字符发送，则会发现这时tcp连接中断，nc发送失败退出。

	hostB$ nc hostA 5555 -p 6666
	a<CR>
	(exit)
	hostB$
	
hostA上的nc监听进程也因为连接中断而退出

	hostA$ nc -l -p 5555
	(exit)
	hostA$

netstat已经观察不到这个tcp连接，而tcpdump此时则捕获了一个新tcp rst包：

	hostA# tcpdump -i eth1 port 5555
	IP hostB.6666 > hostA.5555: Flags [S], seq 750827752, ...
	IP hostA.5555 > hostB.6666: Flags [S.], seq 1191909671, ack 750827753, ...
	IP hostB.6666 > hostA.5555: Flags [.], ack 1, win 115, ...
	IP hostA.5555 > hostB.6666: Flags [R], seq 1191909672, ...

此时tcpkill的输出

	hostA# tcpkill -1 -i eth1 port 5555
	tcpkill: listening on eth1 [port 5555]
	hostB:6666 > hostA:5555: R 1191909672:1191909672(0) win 0
	hostA:5555 > hostB:6666: R 750827755:750827755(0) win 0
	
相信看到这里，已经可以明白tcpkill的工作原理，实际上就是通过双向fake tcp rst包重置目标连接双方的网络连接，和某墙的原理一样。

而之所以tcpkill不会马上中断目标tcp连接，是因为伪造tcp rst包时，需要填入正确的sequence number，这需要通过拦截双方的tcp通信才能实时得到。所以运行tcpkill后，只有目标连接有新tcp包发送/接受才会导致tcp连接中断。

最后分析一下tcpkill第二个参数的具体作用。manpage里的说明比较模糊，只能看出和receive window有关：

> -1...9 Specify the degree of brute force to use in killing a connection. Fast connections may require a higher number in order to land a RST in the moving receive window. Default is 3.

直接看源代码(只有100多行)

	...
	
	int
	main(int argc, char *argv[])
	{
		...
		
		/* 通过libpcap抓取所有符合条件的包，回调函数为tcp_kill_cb */
		pcap_loop(pd, -1, tcp_kill_cb, (u_char *)&sock);

		...
	}
	
	static void
	tcp_kill_cb(u_char *user, const struct pcap_pkthdr *pcap, const u_char *pkt)
	{
		...
		
		/* 只处理tcp包 */
		ip = (struct libnet_ip_hdr *)pkt;
		if (ip->ip_p != IPPROTO_TCP)
			return;
		
		/* 不处理tcp syn/fin/rst包 */
		tcp = (struct libnet_tcp_hdr *)(pkt + (ip->ip_hl << 2));
		if (tcp->th_flags & (TH_SYN|TH_FIN|TH_RST))
			return;
		
        /* 伪造ip包 */
		libnet_build_ip(TCP_H, 0, 0, 0, 64, IPPROTO_TCP,
				ip->ip_dst.s_addr, ip->ip_src.s_addr,
				NULL, 0, buf);

		/* 伪造tcp rst包 */
		libnet_build_tcp(ntohs(tcp->th_dport), ntohs(tcp->th_sport),
				0, 0, TH_RST, 0, 0, NULL, 0, buf + IP_H);
		
		/* fake tcp rst包的sequence number即为抓到的包的ack number */
		seq = ntohl(tcp->th_ack);
		
		...
		
		/* 这里Opt_severity即为tcpkill的第二个参数 */
		win = ntohs(tcp->th_win);
		for (i = 0; i < Opt_severity; i++) {
			ip->ip_id = libnet_get_prand(PRu16);
			seq += (i * win);
			tcp->th_seq = htonl(seq);
	
			libnet_do_checksum(buf, IPPROTO_TCP, TCP_H);
	
            /* 发送伪造的tcp rst包 */
			if (libnet_write_ip(*sock, buf, sizeof(buf)) < 0)
				warn("write_ip");
	
			fprintf(stderr, "%s R %lu:%lu(0) win 0\n", ctext, seq, seq);
		}
	}
	
从上面可以看出，tcpkill的第二个参数，实际上就是沿tcp连接窗口滑动而发送的tcp rst包个数。将这个参数设置较大主要是应对高速tcp连接的情况。

参数的大小从中断tcp连接的原理上没有区别，只是发送rst包数量的差异，通常情况下使用默认值3已经完全没有问题了。所以使用tcpkill时请不要像网络上某些中文教程中一样不适当的使用参数 `-9` 。
