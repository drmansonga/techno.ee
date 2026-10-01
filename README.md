
Lab 4.A — Subnetting and addressing plan

Part 1 — paper (30 min, no calculators). Ten problems: given an address and prefix, find network, broadcast, host range, and usable count; given a requirement ("four subnets of at least 25 hosts from 192.168.4.0/24"), produce the allocation. Peer-mark against the answer key.

Part 2 — the lab's addressing plan (20 min). Each student writes the plan they will use for the rest of the course and commits it to their repo:

Segment	VirtualBox network	IPv4	IPv6 (ULA)	Members

Management	Host-only vboxnet0	192.168.56.0/24	—	Windows host .1, ubuntu .10, rocky .11, router .12

LAN A	Internal labnet-a	10.10.10.0/24	fd00:10::/64	router .1, ubuntu .20

LAN B	Internal labnet-b	10.10.20.0/24	fd00:20::/64	router .1, rocky .20



Lab 4.B — Static addressing on both families

Tasks.

Power off VMs. Add Adapter 3 as Internal Network labnet-a on the Ubuntu VM, labnet-b on the Rocky VM.
Ubuntu, Netplan. /etc/netplan/01-lab.yaml:

yaml
   network:
     version: 2
     ethernets:
       enp0s3: {dhcp4: true}
       enp0s8: {addresses: [192.168.56.10/24]}
       enp0s9:
         addresses: [10.10.10.20/24, "fd00:10::20/64"]


sudo netplan try first — it auto-reverts after 120 seconds if you do not confirm, which is the correct habit for remote network changes and directly foreshadows the Week 7 lockout lesson. Then netplan apply. 3. Rocky, nmcli.

bash
   nmcli con add type ethernet ifname enp0s9 con-name lanb \
     ipv4.method manual ipv4.addresses 10.10.20.20/24 \
     ipv6.method manual ipv6.addresses fd00:20::20/64
   nmcli con up lanb
   nmcli con show lanb | grep -i ipv
Verify on both: ip addr, ip -6 addr, ip link (note the MTU and MAC), ip neigh after pinging.
Prove isolation: from Ubuntu, ping 10.10.10.20 (itself, works) and 10.10.20.20 (Rocky's LAN B address — fails). This failure is the point. Two hosts with addresses but no path between them is exactly the problem Week 5 solves. Have students write down why it fails before the lecture answers it.
Observe ARP: ip -s neigh flush all, then ping a neighbour while running sudo tcpdump -i enp0s9 -n arp in another session. Students see the request and reply.

Deliverable. Committed addressing plan, both config files, and annotated output showing the successful and failed pings with an explanation of the failure.

Pitfalls. Interface naming again — enp0s9 is not guaranteed. Netplan YAML indentation. On Rocky, forgetting nmcli con up after add. Students editing config on Windows and hitting CRLF (§0.7).

Lab 5.A — Build a router

Topology. Three VMs. The router needs three adapters.

<img width="578" height="163" alt="Screenshot 2026-10-01 at 08 10 47" src="https://github.com/user-attachments/assets/b4c514ee-796c-48ce-aae4-9d5bca48df64" />


Tasks.

Create the router VM (or repurpose a minimal clone). Adapters: 1 = NAT, 2 = host-only 192.168.56.12, 3 = internal labnet-a 10.10.10.1/24, 4 = internal labnet-b 10.10.20.1/24.
Enable forwarding, persistently, and prove it survives a reboot (sysctl net.ipv4.ip_forward).
On the Ubuntu VM, add a route to LAN B via the router:
yaml
   routes:
     - to: 10.10.20.0/24
       via: 10.10.10.1

On Rocky: nmcli con mod lanb +ipv4.routes "10.10.10.0/24 10.10.20.1" then nmcli con up lanb. 4. Ping across. When it fails, work the layers bottom-up rather than guessing: is the interface up (ip link), is the address right (ip addr), is there a route (ip route get 10.10.20.20), does the router forward (sysctl), is the packet arriving (tcpdump -i any icmp on the router), is the reply coming back (the return route is the usual culprit). 5. mtr 10.10.20.20 from Ubuntu — students see two hops and can now read a traceroute meaningfully. 6. Repeat the whole exercise for IPv6 with the ULA prefixes.

Checkpoint. Ping succeeds both directions over IPv4 and IPv6, and the student can show the tcpdump output on the router proving the packet transited.

Lab 5.B — DNS

Tasks.

On the router VM, install unbound as a caching recursive resolver. Minimal config: listen on 10.10.10.1 and 10.10.20.1, access-control allowing both LANs, everything else default.
Point the Ubuntu and Rocky VMs at it (Netplan nameservers:, nmcli ipv4.dns). Verify with resolvectl status — not by reading /etc/resolv.conf.
Measure the cache: dig example.com twice, compare the query times in the output. Explain the difference.
Add local names. Either /etc/hosts entries on each node, or (better) a local-data zone in unbound for lab.local covering all three VMs. Discuss why the second scales and the first does not.
Break it deliberately, three ways, and diagnose each: (a) wrong resolver address, (b) resolver reachable but refusing the query via access-control, (c) a stale /etc/hosts entry pointing at the wrong IP. For each, students record which command distinguished this cause from the others.
Capture a DNS lookup: sudo tcpdump -i enp0s9 -n port 53 while running dig. Identify the query and the response.

Deliverable. Working resolver config, evidence of caching (two dig timings), and a short "three failures" table: symptom, distinguishing command, cause, fix.
