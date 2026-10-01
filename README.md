Lab 5.A — Build a router

Topology. Three VMs. The router needs three adapters.

   Windows host
        |  host-only 192.168.56.0/24 (management, all three VMs)
  +-----+-----+
  |  router   |  10.10.10.1  (labnet-a)   10.10.20.1  (labnet-b)
  +-----+-----+
        |                          |
  ubuntu 10.10.10.20        rocky 10.10.20.20

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
