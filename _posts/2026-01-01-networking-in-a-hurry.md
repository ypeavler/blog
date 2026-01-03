---
layout: post
title: "Networking in a Hurry: From ARP to Geneve(Q&A Format)"
categories: Learn in a hurry
author:
- Yuva Peavler
modified_date: 2026-01-01
---

This guide covers networking technologies from the fundamentals to modern cloud-native networking, organized chronologically from basics to advanced concepts. You'll learn everything from the OSI model to Kubernetes networking, including ARP, NAT, VLANs, VXLAN, and Geneve. The guide progresses from foundational concepts (OSI model, Layer 2/3/4) through container networking primitives, then covers network virtualization technologies (VLAN, VXLAN, Geneve) in order of their development.

<div class="video-wrapper" style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%; margin: 2em 0;">
  <iframe src="https://www.youtube.com/embed/cjQLXylqrKM" style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: 0;" allowfullscreen title="Networking Overview Video"></iframe>
</div>

**Test Your Knowledge**
Ready to test your understanding? Take the **[Networking Quiz](https://ypeavler.github.io/blog/2026/01/01/networking-basics-quiz.html)** — 34 questions covering everything from ARP to service mesh.

---

## Part 1: The Fundamentals

### The OSI Model

<a id="what-is-the-osi-model-and-why-do-i-need-to-know-it"></a>

#### Q: What is the OSI model and why do I need to know it?

The <a href="https://en.wikipedia.org/wiki/OSI_model" target="_blank" rel="noopener noreferrer">OSI (Open Systems Interconnection) model</a> divides networking into seven layers. Each layer only communicates with the layers directly above and below it. It's your mental map for understanding how networking works.

<a id="what-are-the-seven-layers-of-the-osi-model"></a>

#### Q: What are the seven layers of the OSI model?

- **Layer 7: Application** — HTTP, DNS, SSH
- **Layer 6: Presentation** — TLS/SSL, Compression
- **Layer 5: Session** — Connection management
- **Layer 4: Transport** — TCP, UDP (Ports)
- **Layer 3: Network** — IP (Routing)
- **Layer 2: Data Link** — Ethernet, MAC (Switching)
- **Layer 1: Physical** — Cables, Radio, Fiber

---

<a id="whats-the-tcpip-model"></a>

#### Q: What's the TCP/IP model?

For practical purposes, we often use the simpler *TCP/IP model* which combines some layers:

- **Application Layer** — HTTP, DNS, SSH
- **Transport Layer** — TCP, UDP
- **Internet Layer** — IP, ICMP
- **Network Access Layer** — Ethernet, Wi-Fi

---

<a id="what-does-l2-over-l3-tunneling-mean"></a>

#### Q: What does "L2 over L3 tunneling" mean?

Normally, Layer 3 packets (IP packets) are encapsulated inside Layer 2 frames (Ethernet frames). This is the standard way networking works: an IP packet gets wrapped in an Ethernet frame with MAC addresses, and the frame is delivered to the next hop.

"L2 over L3 tunneling" reverses this: it wraps an entire Ethernet frame (Layer 2) inside an IP packet (Layer 3). This is what technologies like VXLAN and Geneve do.

**Why is this useful?** It allows you to create virtual Layer 2 networks that span across Layer 3 infrastructure. For example, you can have two VMs in different data centers that appear to be on the same Layer 2 network, even though they're separated by routers and IP networks. The original Ethernet frame (with its MAC addresses) is preserved inside the IP packet, allowing Layer 2 protocols and features to work across the tunnel.

---

### Layer 2: Getting to Your Neighbor

<a id="what-is-layer-2-about"></a>

#### Q: What is Layer 2 about?

Layer 2 is about communication within a *local network segment*—devices that can reach each other without going through a router.

---

<a id="what-is-a-mac-address"></a>

#### Q: What is a MAC address?

Every network interface card (NIC) has a unique 48-bit *MAC address* (Media Access Control), written as six pairs of hex digits like `00:1A:2B:3C:4D:5E`. The first 3 bytes identify the manufacturer (OUI), and the last 3 bytes are unique to the device.

**Examples:**

- `00:50:56:xx:xx:xx` → VMware
- `02:42:xx:xx:xx:xx` → Docker
- `52:54:00:xx:xx:xx` → QEMU/KVM

---

<a id="what-is-arp-and-why-do-i-need-it"></a>

#### Q: What is ARP and why do I need it?

When Host A wants to send a packet to Host B (same subnet), it knows B's IP address but not its MAC address. *ARP (Address Resolution Protocol)* solves this.

---

<a id="how-does-arp-work"></a>

#### Q: How does ARP work?

Think of IP addresses as street addresses and MAC addresses as the actual mailbox. Before you can deliver a letter, you need to know which mailbox (MAC) belongs to that address (IP). ARP is like shouting down the street: "Who lives at 192.168.1.20?" and waiting for the owner to respond with their mailbox number.

<div class="mermaid">
sequenceDiagram
    autonumber
    participant A as Host A<br/>192.168.1.10
    participant SW as Switch
    participant B as Host B<br/>192.168.1.20
    participant C as Host C<br/>192.168.1.30

    Note over A: Need MAC for 192.168.1.20

    A->>SW: ARP Request (Broadcast)<br/>"Who has 192.168.1.20?"

    SW->>B: Broadcast
    SW->>C: Broadcast

    Note over C: Not me, ignore

    Note over B: That's me!

    B->>SW: ARP Reply<br/>"192.168.1.20 = BB:BB:BB:BB:BB:BB"
    SW->>A: Forwarded to A

    Note over A: Cache: 192.168.1.20 → BB:BB:BB:BB:BB:BB

    A->>SW: Send packet<br/>Dst MAC: BB:BB:BB:BB:BB:BB
    SW->>B: Delivered!
</div>

---

<a id="how-do-i-view-my-arp-cache"></a>

#### Q: How do I view my ARP cache?

```bash
$ ip neigh
192.168.1.1 dev eth0 lladdr 00:11:22:33:44:55 REACHABLE
192.168.1.20 dev eth0 lladdr bb:bb:bb:bb:bb:bb STALE
```

---

<a id="what-is-a-switch-and-how-does-it-work"></a>

#### Q: What is a switch and how does it work?

A *switch* is a Layer 2 device that learns which MAC addresses are on which ports by observing traffic. A switch is like a smart mail carrier who learns the neighborhood. When you send a letter, the carrier looks at the return address (source MAC) and remembers "this person lives on Elm Street." When mail arrives for that person, the carrier knows exactly which street to go to, instead of delivering to every house.

---

<a id="how-does-mac-learning-work-on-a-switch"></a>

#### Q: How does MAC learning work on a switch?

<div class="mermaid">
sequenceDiagram
    autonumber
    participant A as Host A<br/>Port 1
    participant SW as Switch
    participant B as Host B<br/>Port 2

    Note over SW: MAC table: empty

    A->>SW: Frame from AA:AA to BB:BB

    Note over SW: Learn: AA:AA on Port 1<br/>Unknown destination → FLOOD

    SW->>B: Frame (flooded)

    B->>SW: Reply from BB:BB to AA:AA

    Note over SW: Learn: BB:BB on Port 2<br/>Forward to Port 1

    SW->>A: Frame delivered

    Note over SW: Future A↔B traffic<br/>is unicast, not flooded
</div>

---

### Layer 3: Getting Across Town

<a id="what-is-layer-3-about"></a>

#### Q: What is Layer 3 about?

Layer 3 is about communication *between networks*—when you need to go beyond your local segment.

---

<a id="what-is-an-ip-address"></a>

#### Q: What is an IP address?

An *IP address* is a unique identifier assigned to each device on a network. There are two versions in use today: IPv4 and IPv6.

**IPv4 (Internet Protocol version 4):**
- A 32-bit number, written as four octets (8 bits each) separated by dots
- Example: `192.168.1.100`
- Each octet ranges from 0-255
- Total address space: 2^32 = 4,294,967,296 addresses (~4.3 billion)
- Format: `xxx.xxx.xxx.xxx` where each xxx is 0-255


**IPv6 (Internet Protocol version 6):**
- A 128-bit number, written as eight groups of four hexadecimal digits separated by colons
- Example: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
- Can be shortened by removing leading zeros: `2001:db8:85a3::8a2e:370:7334`
- Total address space: 2^128 = 340,282,366,920,938,463,463,374,607,431,768,211,456 addresses (~340 undecillion)
- Format: `xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx` where each xxxx is 0-FFFF

**Why do we need IPv6?**

IPv4 address exhaustion is the primary driver. With only ~4.3 billion addresses and billions of devices (computers, phones, IoT devices, servers), we've run out of public IPv4 addresses. This has led to:

1. **NAT (Network Address Translation) overuse**: Multiple devices sharing one public IP, which breaks the end-to-end principle of the internet
2. **Address scarcity**: Organizations paying premium prices for IPv4 address blocks
3. **Complexity**: Multiple layers of NAT making networking harder to troubleshoot

IPv6 solves this by providing:

- **Vast address space**: Enough addresses for every device on Earth (and trillions more)
- **Simplified networking**: No NAT needed—every device can have a globally routable address
- **Better performance**: Simpler packet headers, more efficient routing
- **Built-in security**: IPsec support is mandatory in IPv6
- **Auto-configuration**: Devices can automatically configure their addresses (SLAAC)
- **Better mobile support**: Improved handling of devices moving between networks

**The transition:** While IPv6 is the future, we're in a transition period. Most networks support both (dual-stack), allowing devices to use either protocol. IPv4 will likely remain in use for decades due to legacy systems, but new deployments increasingly prioritize IPv6.

---

<a id="what-is-a-subnet-and-how-does-cidr-notation-work"></a>

#### Q: What is a subnet and how does CIDR notation work?

A *subnet* defines which part of the address is the "network" and which is the "host". In CIDR notation `192.168.1.100/24`:

- Network portion: first 24 bits (192.168.1)
- Host portion: last 8 bits (.100)
- Subnet mask: 255.255.255.0
- Network: 192.168.1.0
- Broadcast: 192.168.1.255
- Total hosts: 254

---

<a id="what-are-common-subnet-sizes"></a>

#### Q: What are common subnet sizes?

| CIDR | Subnet Mask | Use Case | Hosts |
|------|-------------|----------|-------|
| /8 | 255.0.0.0 | Large enterprise (10.x.x.x) | 16 million |
| /16 | 255.255.0.0 | Medium network (172.16.x.x) | 65,534 |
| /24 | 255.255.255.0 | Typical LAN | 254 |
| /32 | 255.255.255.255 | Single host | 1 |

---

<a id="how-do-you-calculate-the-number-of-hosts-in-a-subnet"></a>

#### Q: How do you calculate the number of hosts in a subnet?

The formula is: **2^(host bits) - 2**

- **Host bits** = 32 - CIDR prefix (for IPv4)
- Subtract 2 because the network address (all zeros) and broadcast address (all ones) cannot be assigned to hosts

**Examples:**

**/24 subnet:**

- Host bits: 32 - 24 = 8 bits
- Total addresses: 2^8 = 256
- Usable hosts: 256 - 2 = **254**

**/16 subnet:**

- Host bits: 32 - 16 = 16 bits
- Total addresses: 2^16 = 65,536
- Usable hosts: 65,536 - 2 = **65,534**

**Why not 254 × 254?**

A common misconception is that /16 = 254 × 254 = 64,516. This is incorrect because:

- In a /16 subnet, the host portion is the **last 16 bits** (the last two octets combined)
- This gives us 2^16 = 65,536 total addresses, not 254 × 254
- The 254 × 254 calculation would only apply if we were thinking of it as two separate /24 subnets, which is not how /16 works
- In a /16, all 16 host bits are used together as one address space

**/8 subnet:**

- Host bits: 32 - 8 = 24 bits
- Total addresses: 2^24 = 16,777,216
- Usable hosts: 16,777,216 - 2 = **16,777,214** (often rounded to "16 million")

---

<a id="how-does-a-host-decide-if-a-destination-is-local-or-remote"></a>

#### Q: How does a host decide if a destination is local or remote?

When a host wants to send a packet, it first asks: *"Is the destination on my local network?"* Before sending a letter, you check: "Is this address on my street?" If yes, you just walk over and deliver it yourself (ARP and direct delivery). If no, you drop it in the mailbox for the postal service to handle (send to default gateway/router). You don't need to know the entire postal system—just whether it's local or needs to go through the post office!

---

<a id="what-is-a-router-and-how-does-routing-work"></a>

#### Q: What is a router and how does routing work?

A *router* connects multiple networks. It uses a *routing table* to decide where to send each packet. A router is like a post office sorting facility. When a letter arrives, the postal worker looks at the destination address (IP) and checks the routing table—a big directory that says "letters for 10.0.5.0 go to the downtown post office, letters for 10.0.1.0 go to the local branch." The router doesn't change the address on your envelope (IP stays the same), but it knows which "next post office" (next hop) to send it to.

**Example routing table:**

- `10.0.1.0/24` → eth0 (directly connected)
- `10.0.2.0/24` → eth1 (directly connected)
- `10.0.5.0/24` → via 10.0.2.254 (next hop)
- `0.0.0.0/0` → via 203.0.113.1 (default route)

---

<a id="do-ip-addresses-change-as-packets-traverse-the-network"></a>

#### Q: Do IP addresses change as packets traverse the network?

> **💡 Key Insight**
>
> IP addresses never change as packets traverse the network. Only MAC addresses change at each hop.

**Why IP addresses stay the same:**

IP addresses are **logical addresses** that represent the final destination (and source) of the packet. Think of them as the address written on your envelope—the destination address (10.0.5.100) is where you want the letter to ultimately arrive, and the return address (192.168.1.50) is where it came from. These never change because they represent the actual source and destination hosts.

**Why MAC addresses change:**

MAC addresses are **physical addresses** that represent the immediate next hop. At each router or switch, the packet needs to be delivered to the next device in the path. The MAC address is rewritten to point to the next hop's physical interface.

Think of it like sending a letter from New York to Los Angeles: your envelope has the final destination address written on it (the IP address: "To: 456 Oak Ave, Los Angeles"), which never changes. But at each post office, postal workers add a new routing label (the MAC address) that says "deliver to the next post office's mailroom." These routing labels change at each sorting facility: "Route to Chicago sorting center" → "Route to Denver sorting center" → "Route to LA local post office" → "Deliver to 456 Oak Ave". Each label is specific to the next hop and gets replaced at each facility, while the original addresses on the envelope remain unchanged throughout the journey.

**Example: Packet traveling from Host A to Host B through two routers**

The diagram below shows how a packet travels from Host A (192.168.1.50) to Host B (10.0.5.100) through two routers, demonstrating how MAC addresses change at each hop while IP addresses remain unchanged:

<div class="mermaid">
sequenceDiagram
    autonumber
    participant A as Host A<br/>192.168.1.50
    participant R1 as Router 1
    participant R2 as Router 2
    participant B as Host B<br/>10.0.5.100

    Note over A: Destination: 10.0.5.100<br/>Send to gateway

    Note over A,R1: HOP 1: Host A → Router 1
    A->>R1: Frame<br/>MAC: AA:AA → R1:01<br/>IP: 192.168.1.50 → 10.0.5.100

    Note over R1: Strip frame, lookup route<br/>Next hop: Router 2

    Note over R1,R2: HOP 2: Router 1 → Router 2
    R1->>R2: Frame<br/>MAC: R1:02 → R2:01<br/>IP: 192.168.1.50 → 10.0.5.100

    Note over R2: Strip frame, lookup route<br/>Directly connected

    Note over R2,B: HOP 3: Router 2 → Host B
    R2->>B: Frame<br/>MAC: R2:02 → BB:BB<br/>IP: 192.168.1.50 → 10.0.5.100

    Note over A,B: IP addresses never change!<br/>MAC addresses change at each hop.
</div>

**What happens at each router:**

1. Router receives the Ethernet frame with destination MAC = router's interface MAC
2. Router strips off the Ethernet header (Layer 2)
3. Router examines the IP header (Layer 3) to see the destination IP
4. Router looks up the destination IP in its routing table
5. Router determines the next hop (another router or the final destination)
6. Router uses ARP (if needed) to find the MAC address of the next hop
7. Router creates a **new Ethernet frame** with:

    - Source MAC = router's outgoing interface MAC
    - Destination MAC = next hop's MAC address
    - The original IP packet (unchanged) as the payload


**Why this design matters:**

- **IP addresses** provide end-to-end addressing: the packet knows where it's going and where it came from, regardless of the path taken
- **MAC addresses** provide hop-by-hop delivery: each device only needs to know how to reach the next device, not the entire path
- This separation allows routing to be flexible: if a router goes down, packets can take a different path, but the IP addresses remain the same
- It enables NAT and other middlebox functions: devices in the middle can see and modify the IP packet if needed, but the fundamental source/destination remain

Exception: The one case where IP addresses *do* change is when NAT is involved. NAT devices (like home routers) rewrite the source IP address (and sometimes destination IP) as packets pass through. However, this is a special case of address translation, not normal routing. In normal routing without NAT, IP addresses remain unchanged.

---

<a id="what-is-ttl-and-why-is-it-important"></a>

#### Q: What is TTL and why is it important?

Every IP packet has a *TTL (Time To Live)* field that decrements at each router. If it reaches 0, the packet is dropped. TTL is like a "maximum number of post offices" stamp on your envelope. Every time your letter goes through a post office (router), they stamp it with one less number. If your letter has been through 64 post offices and still hasn't arrived, it's probably lost in a loop somewhere, so the post office throws it away. This prevents letters from bouncing between post offices forever if someone made a routing mistake!

**Example:** Host A (TTL=64) → Router 1 (TTL=63) → Router 2 (TTL=62) → Router 3 (TTL=61) → Host B

---

<a id="what-is-nat-and-how-does-it-work"></a>

#### Q: What is NAT and how does it work?

*NAT (Network Address Translation)* allows multiple devices with private IPs to share a single public IP. NAT is like an apartment building's mailroom. You write a letter with your apartment number (private IP like 192.168.1.10) as the return address, but when it goes out to the world, the mailroom clerk **changes the return address** on the envelope to the building's public address (203.0.113.50) and keeps a note: "Apartment 10's letter is actually from port 40001." When a reply comes back addressed to the building, the clerk looks up their notes and forwards it to your apartment. The outside world never sees your private address!

<div class="mermaid">
sequenceDiagram
    autonumber
    participant A as Host A<br/>192.168.1.10
    participant NAT as NAT Router
    participant WEB as Web Server

    A->>NAT: Request<br/>Src: 192.168.1.10:54321<br/>Dst: 93.184.216.34:80

    Note over NAT: NAT Table:<br/>192.168.1.10:54321 ↔ 203.0.113.50:40001

    NAT->>WEB: Request<br/>Src: 203.0.113.50:40001<br/>Dst: 93.184.216.34:80

    Note over WEB: Server sees<br/>203.0.113.50 (not 192.168.1.10!)

    WEB->>NAT: Response<br/>Dst: 203.0.113.50:40001

    Note over NAT: Lookup NAT table<br/>Forward to 192.168.1.10:54321

    NAT->>A: Response<br/>Dst: 192.168.1.10:54321
</div>

---

<a id="what-are-the-private-ip-ranges"></a>

#### Q: What are the private IP ranges?

**Private IP ranges (RFC 1918):**

- `10.0.0.0/8` — Large enterprises
- `172.16.0.0/12` — Medium networks
- `192.168.0.0/16` — Home/small office

---

<a id="what-is-bgp-border-gateway-protocol"></a>

#### Q: What is BGP (Border Gateway Protocol)?

*BGP (Border Gateway Protocol)* is the routing protocol used to exchange routing information between autonomous systems (ASes) on the internet. It's the protocol that makes the internet work by allowing different networks to learn how to reach each other.

**What BGP does:**

- **Exchanges routes**: Routers running BGP tell each other which IP address ranges (prefixes) they can reach
- **Path selection**: BGP uses attributes (AS path, local preference, etc.) to choose the best path among multiple options
- **Loop prevention**: BGP prevents routing loops by tracking which autonomous systems a route has passed through
- **Policy enforcement**: Network administrators can set policies to prefer certain paths or block certain routes

**BGP in different contexts:**

1. **Internet BGP (eBGP)**: Used between different organizations/ISPs on the public internet. This is what connects the entire internet together. Each organization has an Autonomous System Number (ASN) and advertises their IP ranges to peers.

2. **Internal BGP (iBGP)**: Used within a single organization to distribute routes between routers in the same autonomous system.

3. **Data center/Cloud BGP**: Used in modern data centers and cloud environments to:
   - Advertise pod/service IP ranges to network infrastructure
   - Enable native routing without overlay encapsulation
   - Integrate with cloud provider route tables
   - Support large-scale Kubernetes deployments

**How BGP works (simplified):**
1. Router A advertises: "I can reach 10.244.0.0/16"
2. Router B receives this advertisement and stores it in its routing table
3. Router B can now forward packets destined for 10.244.0.0/16 to Router A
4. Router B may also advertise this route to other routers (depending on policy)
5. If Router A goes down or withdraws the route, Router B removes it and finds an alternative path

**BGP vs static routes:**

- **Static routes**: Manually configured, don't adapt to changes, don't scale
- **BGP**: Dynamic, automatically adapts to network changes, scales to internet size, supports policies

**Further reading:**

- <a href="https://datatracker.ietf.org/doc/html/rfc4271" target="_blank" rel="noopener noreferrer">RFC 4271: A Border Gateway Protocol 4 (BGP-4)</a> — The BGP specification
- <a href="https://datatracker.ietf.org/doc/html/rfc7938" target="_blank" rel="noopener noreferrer">RFC 7938: Use of BGP for Routing in Large-Scale Data Centers</a> — BGP in data centers
- <a href="https://www.ietf.org/rfc/rfc4272.txt" target="_blank" rel="noopener noreferrer">BGP Best Practices</a> — Operational best practices

---

### Linux Networking Primitives

<a id="what-are-the-linux-building-blocks-for-container-networking"></a>

#### Q: What are the Linux building blocks for container networking?

Before Kubernetes can run pods, Linux provides the building blocks for container isolation: network namespaces, veth pairs, bridges, and iptables.

---

<a id="what-is-a-network-namespace"></a>

#### Q: What is a network namespace?

A *network namespace* gives a process its own isolated network stack—its own interfaces, routes, and iptables rules. Each container gets its own namespace, completely isolated from the host and other containers.

---

<a id="how-do-i-list-network-namespaces"></a>

#### Q: How do I list network namespaces?

```bash
# List network namespaces
$ ip netns list
cni-12345678-abcd-1234-abcd-1234567890ab

# Run a command in a namespace
$ ip netns exec cni-12345678 ip addr
```

---

<a id="create-a-netns"></a>

#### Q: Can I create a network namespce manually?
Yes, you can create a network namespace (netns) manually on a Linux system. This is a common way to experiment with the low-level building blocks that container runtimes use to provide network isolation. 

```bash
# Create a ns
$  sudo ip netns add test-netns

# Execute commands inside a network namespace
$  sudo ip netns exec test-netns ip link list  
    lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

# Bring up the lo interface
$ sudo ip netns exec my-netns ip link set lo up
```

---

<a id="create-isolated-network-namespace"></a>

#### Q: How to create isolated network namespace witout sudo?
To create a network namespace without root privileges, you must combine it with a User Namespace. This allows you to map your current unprivileged user to the root user within the isolated environment, granting necessary CAP_NET_ADMIN capabilities to configure that specific network stack.

```bash
unshare -r -n /bin/bash

# Create an annonymous network namespace
# -r (--map-root-user): Maps your current UID to 0 (root) inside the namespace.
# -n (--net): Creates a new, empty network namespace.
# /bin/bash: Starts a new shell session inside these namespaces.
```

*Note: The `ip netns list` command does not list annonymous ns. Instead use `lsns -t net` to see the created ns.*

---


<a id="what-is-a-veth-pair"></a>

#### Q: What is a veth pair?

A *veth pair* is like a virtual Ethernet cable with two ends. One end stays in the host namespace, the other goes into the container. A veth pair is like a mail slot connecting two rooms. When you drop a letter (packet) into the slot in your room (container), it immediately appears in the other room (host namespace). It's a direct, private connection—like having your own dedicated mail chute that no one else can use.

The operational status of a veth pair is linked. If one end is set to DOWN, the entire virtual link is considered down, mirroring the behavior of a physical cable being unplugged

---

<a id="connect-two-ns-with-veth-pair"></a>

#### Q: How to connect two netns with veth pair?
1. Create a veth pair: One end will stay on the host, and the other will go into the namespace.
`sudo ip link add veth-host type veth peer name veth-ns`
2. Move one end into the namespace:
`sudo ip link set veth-ns netns <namespace_name>`
3. Assign IP addresses: Give both ends an IP in the same private subnet (e.g., 10.1.1.0/24).
```bash
  #Host: 
  sudo ip addr add 10.1.1.1/24 dev veth-host
  #Namespace: 
  sudo ip netns exec <name> ip addr add 10.1.1.2/24 dev veth-ns``
```
4. Bring interfaces up:
```
  sudo ip link set veth-host up
  sudo ip netns exec <name> ip link set veth-ns up
```
Access: You can now reach your application by hitting the namespace's IP (10.1.1.2) from the host.

---

<a id="what-is-a-bridge"></a>

#### Q: What is a bridge?

A *bridge* is a virtual Layer 2 switch inside the kernel. It connects all the veth pairs together. A bridge is like a shared mailroom in an apartment building. Each apartment (container) has its own mail slot (veth pair) connecting to the mailroom (bridge). When you send a letter to your neighbor, you drop it in your slot, it arrives in the mailroom, and the mailroom knows which slot belongs to your neighbor and delivers it there. The mailroom (bridge) learns which apartment (container) is connected to which slot (MAC address) by watching the return addresses on letters.

---

<a id="what-is-TAP"></a>

#### Q: What is TAP? When do I need TAP vs vEth?
While veth pairs are the standard for connecting two kernel-level entities (like two network namespaces), TAP interfaces are essential for scenarios where the network traffic must be handled by user-space software. vEth is a linux primitive while a file-based interface for raw frames is universal. 

If veth is a direct pneumatic tube (Kernel-to-Kernel), TAP is a digital scanner (Kernel-to-Software). The usecases below shows different types of software.

**Usecases:**
1. **Virtual Machines (LimaVM, QEMU)**: Provides the "hardware" network card for a VM. The hypervisor reads frames from the TAP device and injects them into the guest OS.
2. **Rootless Networking**: Tools like slirp4netns (used by Podman and Lima's user-v2) use TAP to provide internet access to containers without requiring sudo.
3. **Network Monitoring**: Used to capture and analyze raw traffic for security (IDS/Firewalls) without disrupting the actual flow.


Easiest way to remember this is by using the access method.

| Method |	Primitives Used | 	Why it's used |
|---|---| ---|
| Rootful (containers) |	veth pair + Bridge |	Used by standard Docker/Kubernetes; requires root to create virtual links.|
| Rootful (VMs)	| TAP interface + Bridge	| Used by KVM/Proxmox to link the Kernel to Hypervisor software.|
| Rootless (Unprivileged)	| TAP interface + slirp4netns |	Used for rootless containers; bypasses root requirements by using user-space networking.|


*Note: In Rootless mode: Ip address assigned to the TAP interface is not visible to host and the host cannot ping that ip.*

---

<a id="TAP-connect-to-bridge"></a>

#### Q: Does TAP connect to bridge?

A TAP interface (like vnet0) can be created and manually attached to a Linux Bridge (br0) in the host kernel using administrative privileges.

**A Rootless example**:

Imagine your host machine is the building, but you aren't the landlord—you're just a tenant. You aren't allowed to install new pneumatic tubes (veth) or modify the main sorting table (Linux Bridge). Lets take an example of [**Lima VM Network**](https://lima-vm.io/docs/config/network/user-v2/)

- The "Virtual Office" (The Lima VM): You are running a VM. It needs a network.
- The TAP Slot (Inside the Namespace): Lima creates a TAP interface inside a private namespace. This is your "Digital Mail Slot."
- The Software Clerk (The User-v2 Daemon): Instead of a physical sorting table, there is a Software Clerk (a process running on your host). This clerk has "hands" on the TAP slot.
- The "Virtual Bridge": If you have two Lima VMs, they both have TAP slots. The Software Clerk holds both TAP slots in its hands. When VM1 sends a letter, the clerk reads it from the first TAP slot and manually "tosses" it into the second TAP slot.
  
In this scenario, the "Bridge" is just a piece of logic inside the clerk’s brain (the software code).

---

<a id="what-is-iptables"></a>

#### Q: What is iptables?

*iptables* (and its successor nftables) is how Linux manipulates network traffic. iptables is like a postal inspector with a rulebook. As letters (packets) flow through the post office (Linux kernel), the inspector checks each one against the rules: "Letters to 10.96.0.100? Change the address to 10.244.1.5. Letters from 10.0.0.5? Block them. Letters to port 80? Route them to port 8080 instead." The inspector can rewrite addresses, block letters, or redirect them—all without the sender or receiver knowing.

---

<a id="what-are-the-iptables-chains"></a>

In Linux, the processing of packets follows a strict sequence of tables within each hook. The tables are listed below in their actual order of execution for each hook.

- **PREROUTING**: Applied to all incoming packets before a routing decision is made.
- **INPUT**: Applied to packets destined for a local process/socket.
- **FORWARD**: Applied to packets routed through the host (Pod-to-Pod on different nodes).
- **OUTPUT**: Applied to packets generated by a local process.
- **POSTROUTING**: Applied to all outgoing packets after routing is complete.

**Table Execution Order by Hook**


| Hook (Chain)   | 1st Table | 2nd Table | 3rd Table  | 4th Table     | 5th Table     |
|----------------|-----------|-----------|------------|---------------|---------------|
| PREROUTING     | raw       | mangle    | nat (DNAT) | -             | -             |
| INPUT          | mangle    | filter    | security   | nat (SNAT*)   | -             |
| FORWARD        | mangle    | filter    | security   | -             | -             |
| OUTPUT         | raw       | mangle    | nat (DNAT) | filter        | security      |
| POSTROUTING    | mangle    | nat (SNAT)| -          | -             | -             |

*Note: The nat table in the INPUT chain was introduced in later kernel versions to allow SNAT for traffic destined for the local host.*

**Table Function Definitions**

| Table     | Purpose                                         | Common Targets               |
|-----------|-------------------------------------------------|------------------------------|
| raw       | De-prioritizes connection tracking.            | NOTRACK, DROP                |
| mangle    | Modifies IP header fields (TTL, TOS) or marks packets. | MARK, TOS, TTL         |
| nat       | Changes Source or Destination IP/Ports.        | SNAT, DNAT, MASQUERADE, REDIRECT |
| filter    | The "Firewall." Decisions on packet delivery.  | ACCEPT, DROP, REJECT         |
| security  | Implements SELinux security context marks.     | SECMARK, CONNSECMARK         |

---

<a id="how-does-l2bridge-and-the-iptable-work togtether"></a>

#### Q: How does l2 bridge and iptables work together?
In Linux, Layer 2 (L2) bridges and iptables (which typically operates at Layer 3) work together through a kernel bridge netfilter framework. This interaction allows the system to apply advanced IP-level filtering and NAT to traffic that would otherwise stay purely at the Ethernet frame level. Normally, an L2 bridge forwards traffic based on MAC addresses, bypassing the L3 IP stack where iptables resides. To bridge this gap, Linux uses the br_netfilter kernel module.

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

```

---

<a id="what-is-linux-ipvs-ip-virtual-server"></a>

#### Q: What is Linux IPVS (IP Virtual Server)?

IPVS (IP Virtual Server) is a Linux kernel feature that provides Layer 4 load balancing. It's built into the Linux kernel and operates at the network layer, making it faster and more efficient than iptables for load balancing scenarios.

**How IPVS works:**

- IPVS creates a virtual IP (VIP) that represents a service
- Traffic to the VIP is distributed across multiple real servers (backend pods) using load balancing algorithms
- IPVS maintains a connection table in kernel memory, tracking active connections
- Load balancing happens in the kernel, avoiding the overhead of userspace processing

**IPVS vs iptables for load balancing:**

- **iptables**: Uses NAT rules (DNAT) to rewrite destination IPs. With many services, the iptables rule chain becomes long, and every packet must traverse the chain until it matches. This is O(n) complexity—the more rules, the longer it takes.
- **IPVS**: Uses a hash table for O(1) lookup of backend servers. More efficient for large numbers of services (thousands). Also supports more load balancing algorithms (round-robin, least connections, source hashing, etc.).

**When to use IPVS:**

- Large clusters with many services (1000+)
- Need better performance and lower latency
- Want more load balancing algorithm options
- Can enable IPVS kernel modules (ip_vs, ip_vs_rr, ip_vs_wrr, ip_vs_sh, etc.)

**When to use iptables:**

- Smaller clusters (< 1000 services)
- Simpler setup (no kernel modules needed)
- Default and well-tested option

---


<a id="what-are-cgroups"></a>

#### Q: What are cgroups?

*cgroups (control groups)* are a Linux kernel feature that limits, accounts for, and isolates resource usage (CPU, memory, disk I/O, network bandwidth) of a collection of processes. While cgroups can control network bandwidth and network priority, they are **not a networking primitive** in the same sense as network namespaces, veth pairs, bridges, or iptables.

cgroups are essential for containers to run properly because they provide:

- **Resource limits**: Prevent a container from consuming all CPU or memory
- **Resource accounting**: Track how much CPU, memory, and I/O each container uses
- **Network bandwidth control**: Limit how much network bandwidth a container can use
- **Network priority**: Assign different network priorities to different containers

However, cgroups don't create network connectivity or isolation—that's what network namespaces do. Think of it this way: network namespaces create separate "apartments" (network isolation), while cgroups control how much "utilities" (CPU, memory, bandwidth) each apartment can use. Both are needed for containers to work properly, but they serve different purposes. Without cgroups, containers could consume all system resources and starve other processes, making containers unusable in production environments.

---

### Layer 4: Which Application?

<a id="what-does-layer-4-add-to-networking"></a>

#### Q: What does Layer 4 add to networking?

Layer 4 adds *ports* to identify which application should receive the data.

---

<a id="what-are-ports-and-why-do-we-need-them"></a>

#### Q: What are ports and why do we need them?

A single IP can run many services. *Ports* (0-65535) identify each one. An IP address is like a building address, and ports are like apartment numbers or department mailboxes. When you send mail to "123 Main St, Apartment 80" (IP:port), the mailroom knows to deliver it to the web server department (port 80), not the database department (port 5432). One building (one IP) can have many departments (many ports), each handling different types of mail!

---

<a id="what-are-some-well-known-ports"></a>

#### Q: What are some well-known ports?

- 22 → SSH
- 80 → HTTP
- 443 → HTTPS
- 53 → DNS

---

<a id="what-is-a-5-tuple"></a>

#### Q: What is a 5-tuple?

A connection is uniquely identified by the *5-tuple*:

- Protocol: TCP or UDP
- Source IP
- Source Port
- Destination IP
- Destination Port

---

<a id="whats-the-difference-between-tcp-and-udp"></a>

#### Q: What's the difference between TCP and UDP?

| Aspect | TCP | UDP |
|--------|-----|-----|
| Connection | Connection-oriented (handshake first) | Connectionless (fire and forget) |
| Reliability | Guaranteed delivery, ordering | No guarantees |
| Use case | HTTP, SSH, Database queries | DNS, Video streaming, VXLAN tunnels |
| Overhead | Higher (acknowledgments, retries) | Lower (just send it) |
| Header size | 20+ bytes (with options) | 8 bytes (fixed) |

TCP is like **registered mail with delivery confirmation**. You send a letter, the recipient signs for it and sends back a confirmation card. If you don't get the confirmation, you send another letter. The postal service guarantees your letter arrives in order. UDP is like **regular mail**—you drop it in the mailbox and hope it gets there. It's faster and cheaper, but there's no guarantee. For important documents (web pages, database queries), you use registered mail (TCP). For quick notes where losing one doesn't matter (video streaming, DNS lookups), you use regular mail (UDP).

---

<a id="what-is-the-tcp-three-way-handshake"></a>

#### Q: What is the TCP three-way handshake?

Before TCP can send data, it establishes a connection through a three-way handshake. This ensures both sides are ready to communicate and agree on initial sequence numbers. Think of it like a phone call: you dial (SYN), the other person picks up and says "hello" (SYN-ACK), and you confirm "yes, I can hear you" (ACK). Only then do you start talking.

<div class="mermaid">
sequenceDiagram
    autonumber
    participant C as Client
    participant S as Server

    Note over C,S: TCP Three-Way Handshake
    C->>S: SYN (seq=100)
    S->>C: SYN-ACK (seq=300, ack=101)
    C->>S: ACK (ack=301)
    Note over C,S: Connection established!

    C->>S: Data (seq=101)
    S->>C: ACK (ack=201)
</div>

---

<a id="what-is-the-tcp-connection-termination-process"></a>

#### Q: What is the TCP connection termination process?

TCP uses a four-way handshake to gracefully terminate a connection:

FIN: The sender sends a FIN (finish) to indicate it has no more data to send.
ACK: The receiver acknowledges the FIN.
FIN: The receiver sends its own FIN to indicate it has finished sending data.
ACK: The sender acknowledges the receiver's FIN.

---

<a id="what-is-tcp-sliding-window"></a>

#### Q: What is the TCP sliding window and why we need it?

**Why We Use It**
Without a sliding window, TCP would be "Stop-and-Wait": the sender would send one packet and wait for an acknowledgment (ACK) before sending the next. This would be incredibly slow, especially on high-latency links.
The sliding window solves two problems:
  - Throughput Efficiency: It allows the sender to have multiple packets "in flight" at once, filling the network "pipe."
  - Buffer Protection: It prevents the receiver's memory buffer from overflowing. If the receiver's application is slow (e.g., a slow disk write), the window shrinks to tell the sender to slow down.
  
**How It Works**
The control of the sliding window is a dynamic "handshake" between the receiver and the sender.

**The Receiver's Role (Flow Control)**
The receiver controls the window size through a field in the TCP header called the Receive Window (rwnd).
Advertising: In every ACK sent back to the sender, the receiver includes the current size of its available buffer.
**Zero Window:** If the receiver's buffer is completely full, it sends an ACK with a window size of 0. The sender then stops transmitting and periodically sends "Zero Window Probes" to see if space has opened up.

**The Sender's Role (Congestion Control)**
The sender does not just blindly follow the receiver's advertised window. It maintains its own internal limit called the Congestion Window (cwnd), based on how much the network (routers/switches) can handle.
The Formula: The actual amount of data sent is always min(rwnd, cwnd).

**Scaling (Window Scaling)**
The original TCP specification limited the window size to 65,535 bytes (64 KB). On modern high-speed networks (10Gbps+), this is too small.

**TCP Window Scale Option:** This allows the window to be scaled up to 1 GB.
**Configuration:** On Linux, this is controlled by the sysctl parameter: `net.ipv4.tcp_window_scaling = 1`

**Tuning the Buffers**
While the window slides automatically, you control the maximum potential size of that window by adjusting the Linux network buffer limits:
  ```
  Read Buffer: net.ipv4.tcp_rmem (min, default, max)
  Write Buffer: net.ipv4.tcp_wmem (min, default, max)
  ```
By increasing these values in /etc/sysctl.conf, you allow the sliding window to grow larger, which is essential for high-latency, high-bandwidth connections (like communicating between data centers across continents)

---

## Part 2: Network Virtualization Technologies

### VLAN: Network Segmentation

<a id="what-is-vlan"></a>

#### Q: What is VLAN?

A *VLAN (Virtual Local Area Network)* is a logical network segment created within a physical network. It allows you to group devices together logically, even if they're not physically connected to the same switch. VLANs are identified by a VLAN ID (a number from 1-4094) that is added to Ethernet frames as a tag. Think of VLANs as creating separate "virtual neighborhoods" within the same physical building—devices in VLAN 10 can't directly communicate with devices in VLAN 20, even though they might be connected to the same physical switch, just like people in different apartment buildings on the same street.

<div class="mermaid">
flowchart LR
    subgraph VLAN10["VLAN 10 (Engineering)"]
        A[Host A]
        B[Host B]
    end

    subgraph VLAN20["VLAN 20 (Accounting)"]
        C[Host C]
        D[Host D]
    end

    SW[("Switch")]

    A <--> SW
    B <--> SW
    C <--> SW
    D <--> SW
</div>

---

<a id="what-problem-did-vlans-solve"></a>

#### Q: What problem did VLANs solve?

In the early days, Ethernet was a "flat" network where every device heard everyone else's broadcasts. W. David Sincoskie invented the <a href="https://www.networkworld.com/article/963787/what-is-a-vlan-and-how-does-it-work.html" target="_blank" rel="noopener noreferrer">VLAN at Bellcore in the 1980s</a> to break these large, noisy broadcast domains into smaller, manageable logical groups. The technology was later standardized as IEEE 802.1Q.

---

<a id="how-do-vlans-work"></a>

#### Q: How do VLANs work?

A VLAN adds a *4-byte 802.1Q tag* to the Ethernet frame. The switch reads this tag and only forwards the frame to ports in the same VLAN. Think of VLANs as colored envelopes. When you send a letter in a blue envelope (VLAN 10), the mail carrier (switch) only delivers it to mailboxes that accept blue envelopes. Letters in green envelopes (VLAN 20) go to different mailboxes. Even though all the mailboxes are on the same street (same physical switch), the colored envelopes keep the mail separated—blue letters never mix with green letters.

---

<a id="can-you-walk-through-a-vlan-example"></a>

#### Q: Can you walk through a VLAN example?

When Host A (192.168.10.5) sends to Host B (192.168.10.6) on VLAN 10, the switch reads the 802.1Q tag and forwards the frame only to ports in VLAN 10, ensuring Host C on VLAN 20 never sees the traffic:

<div class="mermaid">
sequenceDiagram
    autonumber
    participant A as Host A<br/>VLAN 10
    participant SW as Switch
    participant B as Host B<br/>VLAN 10
    participant C as Host C<br/>VLAN 20

    Note over A,C: Port 1 & 2: VLAN 10<br/>Port 3: VLAN 20

    A->>SW: Frame with VLAN 10 tag

    Note over SW: Read 802.1Q tag<br/>VLAN ID = 10<br/>Forward to VLAN 10 ports

    SW->>B: Forward frame<br/>(VLAN 10)

    Note over SW,C: Frame NOT sent to C<br/>(Different VLAN)

    Note over A,C: Broadcast isolation!<br/>C never sees A's traffic
</div>

---

<a id="what-were-the-constraints-of-vlans"></a>

#### Q: What were the constraints of VLANs?

- *Physical port binding*: VLANs were tied to the physical switch port. If you moved your desk, a network engineer had to manually reconfigure the switch.
- *The 4,094 ceiling*: With only a 12-bit ID, you could only have 4,094 usable networks—plenty for an office, but a disaster for the upcoming cloud era.

VLANs were like having only 4,094 different envelope colors available. Once you used all the colors, you couldn't create new networks. Also, if you moved to a different building (different switch port), you had to tell the mailroom "I'm now using blue envelopes instead of green," and they had to manually update their records. This didn't work well when people (VMs) were moving constantly!

---

### VXLAN: Network Virtualization

<a id="what-is-vxlan"></a>

#### Q: What is VXLAN?

*VXLAN (Virtual eXtensible Local Area Network)* is a network virtualization technology that encapsulates Layer 2 Ethernet frames inside Layer 3 UDP packets. This creates an "overlay network" that allows VMs and containers to communicate as if they're on the same local network, even when they're on different physical servers or data centers. VXLAN uses a 24-bit Virtual Network Identifier (VNI) to create up to 16.7 million logical networks, far exceeding VLAN's 4,094 limit. The key innovation is that VXLAN decouples the logical network from the physical network infrastructure—VMs can move between physical servers without changing their network identity, and the physical network only sees IP traffic between servers, not the virtual network details.

Think of VXLAN like putting an envelope inside another envelope. You write your letter (original L2 frame with VM's MAC addresses) and put it in an inner envelope addressed to the destination VM. Then you put that inner envelope inside an **outer envelope** addressed to the destination server (VTEP IP address). The postal service (physical network) only looks at the outer envelope and delivers it to the server. The server then opens the outer envelope, takes out the inner envelope, and delivers it to the VM. The postal service never sees what's inside—they just see mail between servers!

In a multi-tenant environment, VXLAN is a cornerstone. It allows different tenants to have their own logically isolated networks (using unique VNIs) that share the same underlying physical infrastructure, preventing tenants from seeing each other's traffic.

<div class="mermaid">
flowchart TB
    subgraph Overlay["OVERLAY NETWORK (Virtual)"]
        direction LR
        VM1["VM1 VNI: 5000"]
        VM2["VM2 VNI: 5000"]
    end

    subgraph Underlay["UNDERLAY NETWORK (Physical)"]
        direction LR
        VTEP1["VTEP 1"]
        R1["Router"]
        R2["Router"]
        VTEP2["VTEP 2"]
    end

    VM1 <-->|"L2 Frame"| VTEP1
    VTEP1 <-->|"UDP Packet"| R1
    R1 <-->|"IP Routing"| R2
    R2 <-->|"UDP Packet"| VTEP2
    VTEP2 <-->|"L2 Frame"| VM2
</div>

---

<a id="what-is-the-vxlan-architecture"></a>

#### Q: What is the VXLAN architecture?

- **Overlay Network (Virtual)**: VMs think they're on the same L2 segment
- **Underlay Network (Physical)**: Physical network routes between VTEPs (VXLAN Tunnel End Points)

---

<a id="how-many-networks-does-vxlan-support"></a>

#### Q: How many networks does VXLAN support?

The physical switches just saw traffic between servers, while the VMs felt like they're on one giant, *16.7-million-segment* logical switch (thanks to the 24-bit VNI: 2^24 = 16,777,216 possible networks).

---

<a id="can-you-walk-through-a-vxlan-example-flow"></a>

#### Q: Can you walk through a VXLAN example flow?

The diagram below shows the complete VXLAN encapsulation and decapsulation process:

<div class="mermaid">
sequenceDiagram
    autonumber
    participant VM1 as VM1
    participant VTEP1 as VTEP1
    participant NET as IP Network
    participant VTEP2 as VTEP2
    participant VM2 as VM2

    Note over VM1,VM2: VMs on same L2 segment

    VM1->>VTEP1: L2 Frame to VM2

    Note over VTEP1: Lookup: VM2 → VTEP2<br/>VNI: 5000

    Note over VTEP1,VTEP2: ENCAPSULATION
    VTEP1->>NET: UDP Packet<br/>Outer: 10.0.0.1 → 10.0.0.2<br/>VNI: 5000<br/>Inner: L2 Frame

    Note over NET: Physical network<br/>sees only IP traffic

    NET->>VTEP2: Routed packet

    Note over VTEP2: DECAPSULATION<br/>Extract L2 frame

    VTEP2->>VM2: L2 Frame to VM2

    Note over VM1,VM2: VM2 receives frame<br/>as if on same switch
</div>

---

<a id="why-are-they-called-tunnels"></a>

#### Q: Why are they called tunnels?

They are called tunnels because they create a private, direct-path "shortcut" for your data through an existing network, similar to how a physical tunnel allows a car to pass through a mountain instead of driving over every peak. In networking, a "tunnel" isn't a physical wire; it is a logical path created by *encapsulation*.

Here's how it works: You write a letter to your friend (original L2 frame with VM MAC addresses). You put it in an inner envelope addressed to your friend's apartment (destination VM). Then you put that inner envelope inside an **outer envelope** addressed to your friend's building (VTEP IP address). The outer envelope has a special label (VNI) that says "Building 5000" so the receiving building knows which floor to deliver it to. The postal service (physical network) only looks at the address on the **outer envelope**. They see "Deliver to Building 10.0.0.2" and route it there. They have no idea there's another envelope inside, or that it's really meant for someone in apartment 192.168.10.6. When the letter arrives at Building 10.0.0.2, the mailroom (VTEP) opens the outer envelope, reads the VNI label ("Building 5000"), and delivers the inner envelope to the correct apartment (VM). Your friend receives the letter as if you sent it directly—they never see the outer envelope! VXLAN is the "outer envelope," and the VTEP (VXLAN Tunnel End Point) is the "building's mailroom" that handles the envelope wrapping and unwrapping.

---

<a id="how-do-vteps-discover-each-other"></a>

#### Q: How do VTEPs discover each other?

VXLAN requires a *control plane* to map VM MAC addresses to VTEP IP addresses. Common approaches:

| Method | Description |
|--------|-------------|
| Multicast | VTEPs join multicast groups per VNI. Broadcast ARP requests are sent via multicast. Simple but requires multicast support in underlay. |
| BGP-EVPN | BGP extensions for Ethernet VPN (RFC 7432). VTEPs exchange MAC/IP routes via BGP. Used in large-scale deployments (Cisco ACI, Juniper). |
| Centralized Controller | SDN controller (e.g., VMware NSX, OpenStack Neutron) maintains MAC-to-VTEP mappings. VTEPs query controller for unknown destinations. |
| Distributed Database | etcd or similar stores MAC-to-VTEP mappings. Used by container networking plugins. |

---

<a id="why-does-vxlan-use-udp"></a>

#### Q: Why does VXLAN use UDP?

VXLAN uses *UDP* (User Datagram Protocol) as its transport protocol for several important reasons:

1. **Reliability is handled at a higher layer**: The inner Ethernet frame already contains TCP/IP traffic, which provides its own reliability mechanisms. If a TCP packet inside the VXLAN tunnel is lost, TCP will retransmit it. Adding TCP reliability at the tunnel level would create redundant retransmissions and actually hurt performance.

2. **Lower overhead**: UDP has a fixed 8-byte header compared to TCP's 20+ byte header (which can grow with options). For tunnel traffic that may carry thousands of packets per second, this overhead reduction matters significantly.

3. **Hardware offloading**: Modern network interface cards (NICs) can offload UDP encapsulation/decapsulation to hardware, improving performance. TCP's stateful nature makes hardware offloading more complex and less efficient.

4. **No connection state**: UDP is connectionless, meaning there's no connection establishment (three-way handshake) or teardown overhead. This is crucial for tunnel traffic where you want to forward packets as quickly as possible without maintaining connection state.

5. **Avoids TCP-in-TCP problems**: If VXLAN used TCP, you'd have TCP inside TCP. This creates problems like:

    - Head-of-line blocking: If one TCP segment is lost, all subsequent segments wait
    - Congestion control conflicts: Inner and outer TCP connections compete
    - Retransmission storms: Both layers trying to retransmit the same data

---

<a id="what-are-the-mtu-and-fragmentation-considerations-for-vxlan"></a>

#### Q: What are the MTU and fragmentation considerations for VXLAN?

VXLAN encapsulation adds approximately 50 bytes to each packet:

- Outer Ethernet: 14 bytes
- Outer IP: 20 bytes
- Outer UDP: 8 bytes
- VXLAN header: 8 bytes
- **Total overhead: ~50 bytes**

If the underlay MTU is 1500 bytes (standard Ethernet), the effective overlay MTU becomes 1450 bytes. Packets larger than this will be fragmented, causing performance degradation.

---

<a id="how-do-i-avoid-vxlan-fragmentation"></a>

#### Q: How do I avoid VXLAN fragmentation?

> **💡 Avoiding VXLAN Fragmentation**
>
> Configure underlay MTU to 1550+ bytes (jumbo frames) to avoid fragmentation, or reduce overlay MTU to 1450 bytes.

> **⚠️ Operator smell for MTU issues**
>
> Cross-node traffic works for small payloads but gRPC/HTTPS calls with larger bodies RST or time out. Quick test: `ping -M do -s 1472 <remote-node-ip>`; if it fails, drop pod MTU to 1450 or raise underlay MTU.

---

<a id="what-are-the-constraints-of-vxlan"></a>

#### Q: What are the constraints of VXLAN?

- *Fixed 8-byte header*: No room for custom metadata beyond the VNI
- *Limited extensibility*: Can't carry security policies or telemetry inline
- *Control plane dependency*: Requires additional infrastructure for MAC-to-VTEP discovery

---

### Geneve: Extensible Network Virtualization

<a id="what-problem-did-geneve-solve"></a>

#### Q: What problem did Geneve solve?

As we moved into containers and cloud-native platforms, even VXLAN started to show its age. Modern platforms needed to carry more than just a "Network ID"—they needed to carry security policies, telemetry, and "who is talking to whom".

---

<a id="what-is-geneve"></a>

#### Q: What is Geneve?

<a href="https://datatracker.ietf.org/doc/html/rfc8926" target="_blank" rel="noopener noreferrer">Geneve (Generic Network Virtualization Encapsulation)</a> arrived to solve the "fixed header" problem of VXLAN. Its extensible design allows developers to add custom data (Type-Length-Value options) to every packet, which is critical for the complex routing and security required by modern SDN platforms like VMware NSX and cloud-native networking solutions.

Geneve is like VXLAN's envelope-inside-envelope, but with **sticky notes attached to the outer envelope**. You still put your letter (original packet) in an inner envelope, then put that in an outer envelope. But now you can attach metadata stickers to the outer envelope: "Security Policy: Allow-123", "Source: frontend-workload", "Telemetry: latency-tracked". The receiving building (VTEP) reads these stickers before opening the envelope, so it knows how to handle the letter—check security permissions, log metrics, route based on identity. VXLAN's outer envelope was blank except for the address; Geneve's outer envelope is covered in useful information!

For enterprise Kubernetes multi-tenancy, Geneve's extensible TLV options become crucial for enforcing fine-grained network policies and carrying tenant-specific metadata, allowing a single underlying network to enforce diverse security rules for multiple isolated tenants.

**Key difference from VXLAN:** VXLAN has a fixed 8-byte header, while Geneve has a variable-length header (8+ bytes) that can include TLV options for extensibility. This allows Geneve to carry metadata like security policies and telemetry inline with each packet.

---

<a id="what-are-examples-of-geneve-tlv-options"></a>

#### Q: What are examples of Geneve TLV options?

Geneve TLV options can carry various types of metadata. Common examples include: **Security Policy ID** (Class 0x0102, Type 1) containing policy identifiers like "policy-xyz-123"; **Telemetry Data** (Class 0x0103, Type 2) with metrics such as "latency=5ms, hop=3"; and **Source Identity** (Class 0x0104, Type 3) identifying workloads like "workload=frontend-abc". These options allow the network to enforce security policies and collect observability data at the packet level without requiring separate control plane messages.

---

<a id="can-you-show-an-example-of-geneve-with-security-metadata"></a>

#### Q: Can you show an example of Geneve with security metadata?

In cloud-native environments, Geneve TLV options carry security policies and source identity. When the frontend workload sends a packet, it's like writing a letter and putting it in an inner envelope. The SDN controller (like a security guard) checks the sender's ID, looks up the security policy, and attaches stickers to the outer envelope: "From: frontend-workload", "Policy: allow-frontend-to-backend", "Security Level: High". When the letter arrives at the destination building, the security guard there reads the stickers, verifies "Yes, frontend is allowed to talk to backend," and only then opens the envelope and delivers it. If the stickers said "Deny," the letter would be rejected without even opening it!

<div class="mermaid">
sequenceDiagram
    autonumber
    participant VM1 as VM1
    participant Node1 as Node1
    participant NET as Network
    participant Node2 as Node2
    participant VM2 as VM2

    VM1->>Node1: Packet to VM2

    Note over Node1: SDN: Lookup policy<br/>Add identity to Geneve

    Note over Node1,Node2: GENEVE ENCAPSULATION
    Node1->>NET: Geneve packet<br/>VNI: 5000<br/>TLV: identity, policy<br/>Inner: packet
    NET->>Node2: Routed

    Note over Node2: SDN: Read TLV<br/>Verify policy ✅<br/>Decapsulate

    Node2->>VM2: Packet delivered

    Note over VM1,VM2: Policy enforced<br/>via Geneve metadata
</div>

---

<a id="what-are-the-mtu-considerations-for-geneve"></a>

#### Q: What are the MTU considerations for Geneve?

Geneve overhead is variable due to TLV options:

- Base overhead: ~38 bytes (Ethernet + IP + UDP + Geneve base header)
- TLV options: 0-252 bytes (variable)
- **Total overhead: 38-290 bytes**

Configure underlay MTU accordingly. For example, with 100 bytes of TLV options, you need at least 1600 bytes MTU to avoid fragmentation.

---

<a id="what-are-the-constraints-of-geneve"></a>

#### Q: What are the constraints of Geneve?

- *Processing overhead*: Variable-length options require more parsing than fixed headers
- *Hardware support*: Older NICs may not offload Geneve efficiently, especially with TLV options
- *Complexity*: TLV parsing adds CPU overhead compared to VXLAN's simple header

---




## References

### Standards and RFCs

- RFC 826: Ethernet Address Resolution Protocol (ARP)
- RFC 791: Internet Protocol (IP)
- RFC 793: Transmission Control Protocol (TCP)
- RFC 768: User Datagram Protocol (UDP)
- RFC 1918: Address Allocation for Private Internets
- RFC 7348: Virtual eXtensible Local Area Network (VXLAN)
- RFC 8926: Geneve: Generic Network Virtualization Encapsulation
- RFC 7432: BGP MPLS-Based Ethernet VPN (BGP-EVPN)
- IEEE 802.1Q: Virtual Bridged Local Area Networks

---

*This article is part of the "Learning in a Hurry" series, designed to help engineers quickly understand complex technical concepts through analogies and practical examples.*


<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>
  mermaid.initialize({
    startOnLoad: true,
    theme: 'base',
    themeVariables: {
      'darkMode': true,
      'background': '#1c2128',     // Main diagram canvas color
      'primaryColor': '#2d333b',    // Node background
      'primaryTextColor': '#adbac7',// Main text color
      'primaryBorderColor': '#444c56',
      'lineColor': '#444c56',
      'secondaryColor': '#316dca',  // Accents for specific nodes
      'tertiaryColor': '#1c2128'
    }
  });
</script>