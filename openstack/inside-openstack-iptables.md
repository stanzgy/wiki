# Openstack iptables structure 

This page shows openstack iptables structure in a more visual and human-readable way.

## table filter

	-->INPUT
		-j nova-compute-INPUT				# empty by default
		-j nova-network-INPUT				# empty by default
		-j nova-api-INPUT

		-->nova-api-INPUT
			-d {nova-ip-address}/32 -p tcp -m tcp --dport {nova-api-port} -j ACCEPT
		
	-->FORWARD
		-j nova-filter-top					# shared by all vm instances
		-j nova-compute-FORWARD				# empty by default
		-j nova-network-FORWARD
		-j nova-api-FORWARD					# empty by default

		-->nova-filter-top					# shared by both FORWARD and OUTPUT chain
			-j nova-compute-local
			-j nova-network-local			# empty by default
			-j nova-api-local				# empty by default

			-->nova-compute-local			# instance dedicated rules
				-d {inst-fixed-ip}/32 -j nova-compute-inst-{inst-id}
				-d {inst-fixed-ip}/32 -j nova-compute-inst-{inst-id}
				...

				-->nova-compute-inst-{inst-id}							# default vm instance rules
					-m state --state INVALID -j DROP					# drop invalid packet
					-m state --state RELATED,ESTABLISHED -j ACCEPT		# accept related and established packet
					-j nova-compute-provider							# empty by default
					-s {inst-fixed-ip}/32 -p udp -m udp --sport 67 --dport 68 -j ACCEPT
					  # insert our ipset rules here
					  # -m set --match-set {inst-set-name} src -j ACCEPT
					-p icmp -j ACCEPT
					-p tcp -m tcp --dport 22 -j ACCEPT					# 22 is ssh default port
					-j nova-compute-sg-fallback

					-->nova-compute-sg-fallback
						-j DROP											# drop all rest unmatched packets

					   
		-->nova-network-FORWARD				# accept forward packet via network bridge
			-i br100 -j ACCEPT
			-o br100 -j ACCEPT

	-->OUTPUT
		-j nova-filter-top
		-j nova-compute-OUTPUT				# empty by default
		-j nova-network-OUTPUT				# empty by default
		-j nova-api-OUTPUT					# empty by default


## table nat

	-->PREROUTING
		-j nova-compute-PREROUTING			# empty by default
		-j nova-network-PREROUTING
		-j nova-api-PREROUTING				# empty by default

		-->nova-network-PREROUTING
			-d 169.254.169.254/32 -p tcp -m tcp --dport 80 -j DNAT --to-destination {metadata-host-ip}:{meta-host-port}		# metadata host dnat
			-d {inst-floating-ip}/32 -j DNAT --to-destination {inst-fixed-ip}
			-d {inst-floating-ip}/32 -j DNAT --to-destination {inst-fixed-ip}
			...

	-->OUTPUT
		-j nova-compute-OUTPUT				# empty by default
		-j nova-network-OUTPUT				# empty by default
		-j nova-api-OUTPUT					# empty by default
		 
	-->POSTROUTING
		-j nova-compute-POSTROUTING			# empty by default
		-j nova-network-POSTROUTING
		-j nova-api-POSTROUTING				# empty by default
		-j nova-postrouting-bottom

		-->nova-network-POSTROUTING
			-s {inst-fixed-ip-range} -d {metadata-host-ip} -j ACCEPT
			-s {inst-fixed-ip-range} -d {dmz-ip-range} -j ACCEPT
			-s {inst-fixed-ip-range} -d {inst-fixed-ip-range} -m conntrack ! --ctstate DNAT -j ACCEPT

		-->nova-postrouting-bottom			# perform snat here
			-j nova-compute-snat
			-j nova-network-snat
			-j nova-api-snat
			
			-->nova-compute-snat
				-j nova-compute-float-snat	# empty by default

			-->nova-network-snat
				-j nova-network-float-snat
				-s {inst-fixed-ip-range} -j SNAT --to-source {routing-source-ip}

				-->nova-network-float-snat
					-s {inst-fixed-ip} -j SNAT --to-source {inst-floating-ip}
					-s {inst-fixed-ip} -j SNAT --to-source {inst-floating-ip}
					...

			-->nova-api-snat
				-->nova-api-float-snat		# empty by default
