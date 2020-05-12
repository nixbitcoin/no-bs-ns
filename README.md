# No BullShit NameSpaces  

Guides and scripts on how to create network namespaces without bullshit. No MACVLAN, no TAP, no Open vSwitch, no need to know physical interface names. Just `iproute2` (built-in to all Linux distros) and a couple commands.

Network namespaces are connected to a bridge interface via veth paris. Traffic leaves the bridge interface into the default network namespace and is automatically routed via NAT. Everything, including DNS, JustWorks™

Created as notes and educational material for the upcoming nix-bitcoin network namespace refactoring.

Setup in 10 easy steps
---

1. Create network namespaces

	```console
	# ip netns add namespace1
	# ip netns add namespace2
	```

2. Create veth pairs, these act like tubes that transport your traffic from "start" (ex. veth1) to "end" (ex. br-veth1) and vice-versa.

	```console
	# ip link add veth1 type veth peer name br-veth1
	# ip link add veth2 type veth peer name br-veth2
	```

3. Associate veth pair "start" (ex. veth1) with namespace

	```console
	# ip link set veth1 netns namespace1
	# ip link set veth2 netns namespace2
	```

4. Give veth pair "start" (ex. veth1) IPv4 address in namespace

	```console
	# ip netns exec namespace1 ip addr add 172.18.0.11/24 dev veth1
	# ip netns exec namespace2 ip addr add 172.18.0.12/24 dev veth2
	```

	I like to use the 172's for this, because they are not commonly used and therefore don't interfere with my local network.

5. Create bridge

	```console
	# ip link add name br1 type bridge
	# ip link set br1 up
	```

6. Turn everything on

	```console
	# ip link set br-veth1 up
	# ip link set br-veth2 up
	# ip netns exec namespace1 ip link set veth1 up
	# ip netns exec namespace2 ip link set veth2 up
	```

7. Associate veth pair "end" (ex. br-veth1) with bridge (ex. br1)

	```console
	# ip link set br-veth1 master br1
	# ip link set br-veth2 master br1
	```

8. Give bridge IPv4 address

	```console
	# ip addr add 172.18.0.10/24 brd + dev br1
	```

	If you lose your ssh connection at this point, it probably has something to with 172.18.0.10/24 interfering with your local network.

9. Give all namespaces default gateway route

	```console
	# ip -all netns exec ip route add default via 172.18.0.10
	```

10. Set up iptables and enable IPv4 ip forwarding

	```console
	# iptables \
		  -t nat \
		  -A POSTROUTING \
		  -s 172.18.0.0/24 \
		  -j MASQUERADE
	# sysctl -w net.ipv4.ip_forward=1
	```

Usage example: bitcoind and bitcoin-cli in two different namespaces
---

1. Make Tor listen on bridge address

	In `/etc/tor/torrc`

	```
	SocksPort 172.18.0.10:9050
	```

2. Restart Tor

	```console
	# systemctl restart tor
	```

2. Edit bitcoin.conf

	Make it look something like this

	```
	daemon=1
	server=1
	proxy=172.18.0.10:9050
	rpcbind=172.18.0.11
	rpcallowip=172.18.0.12
	```

3. Start `bitcoind`

	```console
	# ip netns exec namespace1 sudo -u <BITCOINUSER> bitcoind
	```

4. Run `bitcoin-cli`

	```console
	# ip netns exec namespace2 sudo -u <BITCOINUSER> bitcoin-cli -rpcconnect=172.18.0.11 -getinfo
	```

Resources
---

1. [Using network namespaces and a virtual switch to isolate servers by cirowrc](https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/)
2. [Introducing Linux Network Namespaces by Scott Lowe](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)
3. [Introduction to Linux interfaces for virtual networking by Red Hat](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/#macvlan)
4. [Bridge vs Macvlan by Hi Cube](https://hicu.be/bridge-vs-macvlan)
