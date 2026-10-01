# Lab 4.A — Subnetting and addressing plan

## Part 1 — paper exercise (30 min, no calculators)

Ten problems: given an address and prefix, find:
- network address
- broadcast address
- host range
- usable host count

Also given a requirement such as "four subnets of at least 25 hosts", determine a valid addressing plan.

This is practice for subnetting logic and address calculation without relying on a calculator.

## Part 2 — the lab's addressing plan (20 min)

Each student writes the plan they will use for the rest of the course and commits it to their repo.

| Segment | VirtualBox network | IPv4 | IPv6 (ULA) | Members |
|---|---|---|---|---|
| Management | Host-only vboxnet0 | 192.168.56.0/24 | — | Windows host .1, ubuntu .10, rocky .11, router .12 |
| LAN A | Internal labnet-a | 10.10.10.0/24 | fd00:10::/64 | router .1, ubuntu .20 |
| LAN B | Internal labnet-b | 10.10.20.0/24 | fd00:20::/64 | router .1, rocky .20 |

---

# Lab 4.B — Static addressing on both families

## Tasks

1. Power off VMs.
2. Add Adapter 3 as Internal Network labnet-a on the Ubuntu VM.
3. Add Adapter 3 as Internal Network labnet-b on the Rocky VM.

### Ubuntu: Netplan

File: `/etc/netplan/01-lab.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s3: {dhcp4: true}
    enp0s8: {addresses: [192.168.56.10/24]}
    enp0s9:
      addresses: [10.10.10.20/24, "fd00:10::20/64"]
```

Apply it with:

```bash
sudo netplan try first
```

This command auto-reverts after 120 seconds if you do not confirm. That is the correct habit for remote network changes and directly foreshadows the Week 7 lockout lesson.

### Rocky: NetworkManager

```bash
nmcli con add type ethernet ifname enp0s9 con-name lanb \
  ipv4.method manual ipv4.addresses 10.10.20.20/24 \
  ipv6.method manual ipv6.addresses fd00:20::20/64
nmcli con up lanb
nmcli con show lanb | grep -i ipv
```

### Verify on both systems

```bash
ip addr
ip -6 addr
ip link
ip neigh
```

Note the MTU and MAC address, and inspect the neighbor table after pinging.

### Prove isolation

From Ubuntu, test:

```bash
ping 10.10.10.20
ping 10.10.20.20
```

Expected result:
- `10.10.10.20` (itself) succeeds
- `10.10.20.20` (Rocky's LAN B address) fails

This failure is the point. Two hosts can have addresses but no route between them, which exactly demonstrates isolation.

### Observe ARP

```bash
sudo ip -s neigh flush all
```

Then, in another session:

```bash
sudo tcpdump -i enp0s9 -n arp
```

Then ping a neighbor while watching the capture and observe the ARP request and reply.

## Deliverable

- committed addressing plan
- both config files
- annotated output showing the successful and failed pings
- explanation of the failure

## Pitfalls

- Interface naming again — `enp0s9` is not guaranteed
- Netplan YAML indentation
- On Rocky, forgetting `nmcli con up` after `add`
- Students editing config on Windows and hitting CRLF (`§0.7`)

---

# Lab 5.A — Build a router

## Topology

Three VMs. The router needs three adapters.

<img width="578" height="163" alt="Screenshot 2026-10-01 at 08 10 47" src="https://github.com/user-attachments/assets/b4c514ee-796c-48ce-aae4-9d5bca48df64" />

## Tasks

Create the router VM (or repurpose a minimal clone). Adapter layout:

1. NAT
2. Host-only: `192.168.56.12`
3. Internal network: `labnet-a` with `10.10.10.1/24`
4. Internal network: `labnet-b` with `10.10.20.1/24`

### Enable forwarding and persist it

```bash
sysctl net.ipv4.ip_forward
```

Enable forwarding persistently, then prove it survives a reboot.

Check it again after reboot:

```bash
sysctl net.ipv4.ip_forward
```

### Ubuntu route to LAN B

Add a route via the router:

```yaml
routes:
  - to: 10.10.20.0/24
    via: 10.10.10.1
```

### Rocky route to LAN A

```bash
nmcli con mod lanb +ipv4.routes "10.10.10.0/24 10.10.20.1"
nmcli con up lanb
```

### Ping across

Test connectivity in both directions.

If it fails, debug from the bottom up instead of guessing:
- Is the interface up?
- Is the route present?
- Is the router forwarding packets?
- Is there a firewall or policy issue?

### Checkpoint

Ping succeeds both directions over IPv4 and IPv6, and the student can show the `tcpdump` output on the router proving the packet transited.

---

# Lab 5.B — DNS

## Tasks

On the router VM, install `unbound` as a caching recursive resolver.

Minimal configuration:
- listen on `10.10.10.1` and `10.10.20.1`
- allow access from both LANs
- leave everything else at default

Point the Ubuntu and Rocky VMs at it using:
- Netplan `nameservers:`
- `nmcli ipv4.dns`

Verify with:

```bash
resolvectl status
```

Do not rely only on `/etc/resolv.conf`.

### Measure the cache

Run:

```bash
dig example.com
dig example.com
```

Compare the query times in the output. Explain the difference.

### Add local names

Choose either:
- `/etc/hosts` entries on each node
- better: a local-data zone in `unbound` for `lab.local` covering all three VMs

Discuss why the second scales and the first does not.

### Break it deliberately

Break the setup in three ways and diagnose each:

1. Wrong resolver address
   - resolver is unreachable or misconfigured
2. Resolver reachable but refusing the query via access-control
   - the service is up but blocked by policy
3. Stale `/etc/hosts` entry pointing at the wrong IP
   - cache or local override is incorrect

### Capture a DNS lookup

Run:

```bash
sudo tcpdump -i enp0s9 -n port 53
```

while executing:

```bash
dig example.com
```

Identify the query and the response.

## Deliverable

- working resolver config
- evidence of caching from two `dig` timings
- short "three failures" table with:
  - symptom
  - distinguishing command
  - cause
  - fix

---

# Final summary

These labs build a working lab network with:
- subnetting and addressing
- static IPv4 and IPv6 configuration
- routing between networks
- DNS resolution and caching
- troubleshooting common networking failures
