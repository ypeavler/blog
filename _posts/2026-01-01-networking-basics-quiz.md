---
layout: page
title: Networking in a hurry - Quiz
---

Test your knowledge of networking fundamentals, from the OSI model to Kubernetes service mesh.

<style>
.nq-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: .4rem 0;
  margin-bottom: .6rem;
  border-bottom: 1px solid var(--md-default-fg-color--lightest);
  font-size: .65rem;
  color: var(--md-default-fg-color--light);
}
.nq-header strong { color: var(--md-primary-fg-color); }
.nq-score-correct { color: #2da44e; font-weight: 600; }
.nq-score-wrong { color: #cf222e; font-weight: 600; }

.nq-card { display: none; }
.nq-card.active { display: block; min-height: 420px; }

.nq-question { font-size: .7rem; line-height: 1.5; margin-bottom: .6rem; }

.nq-options { display: flex; flex-direction: column; gap: .3rem; }
.nq-opt { position: relative; }
.nq-opt input { position: absolute; opacity: 0; pointer-events: none; }
.nq-opt label {
  display: flex; gap: .4rem; padding: .4rem .6rem;
  background: var(--md-code-bg-color);
  border: 1px solid var(--md-default-fg-color--lightest);
  border-radius: .2rem; cursor: pointer;
  font-size: .65rem; line-height: 1.45;
  transition: border-color .15s;
}
.nq-opt label:hover { border-color: var(--md-primary-fg-color); }
.nq-opt-letter { font-weight: 600; color: var(--md-primary-fg-color); min-width: .8rem; }
.nq-opt-text { flex: 1; }
.nq-opt.correct label { border-color: #2da44e; background: rgba(45,164,78,.08); }
.nq-opt.correct .nq-opt-letter { color: #2da44e; }
.nq-opt.wrong label { border-color: #cf222e; background: rgba(207,34,46,.08); }
.nq-opt.wrong .nq-opt-letter { color: #cf222e; }
.nq-opt.disabled label { cursor: default; opacity: .5; }
.nq-opt.disabled.correct label, .nq-opt.disabled.wrong label { opacity: 1; }

.nq-opt-exp {
  display: none;
  margin-top: .3rem;
  padding: .4rem .6rem;
  font-size: .6rem;
  line-height: 1.45;
  border-radius: .2rem;
}
.nq-opt-exp.visible { display: block; }
.nq-opt-exp.correct { background: rgba(45,164,78,.06); border-left: 2px solid #2da44e; }
.nq-opt-exp.wrong { background: rgba(207,34,46,.06); border-left: 2px solid #cf222e; }
.nq-opt-exp-title { font-weight: 600; margin-bottom: .2rem; }
.nq-opt-exp.correct .nq-opt-exp-title { color: #2da44e; }
.nq-opt-exp.wrong .nq-opt-exp-title { color: #cf222e; }
.nq-opt-exp-correct { margin-top: .3rem; padding-top: .3rem; border-top: 1px solid var(--md-default-fg-color--lightest); }
.nq-learn-more {
  display: inline-block; margin-top: .3rem; font-size: .55rem;
  color: var(--md-primary-fg-color) !important; text-decoration: none;
}
.nq-learn-more:hover { text-decoration: underline; }

.nq-hint { margin-top: .6rem; }
.nq-hint-btn {
  background: none; border: none; color: var(--md-default-fg-color--light);
  font: inherit; font-size: .6rem; cursor: pointer; padding: 0;
}
.nq-hint-btn:hover { color: var(--md-primary-fg-color); }
.nq-hint-text {
  display: none; margin-top: .3rem; padding: .4rem .5rem;
  background: rgba(var(--md-primary-fg-color--rgb),.05);
  border-left: 2px solid var(--md-primary-fg-color);
  font-size: .6rem; line-height: 1.45;
}
.nq-hint-text.visible { display: block; }

.nq-nav {
  display: flex; justify-content: space-between; gap: .6rem;
  margin-top: .8rem; padding-top: .6rem;
  border-top: 1px solid var(--md-default-fg-color--lightest);
}
.nq-nav-btn {
  padding: .35rem .75rem; font: inherit; font-size: .6rem; font-weight: 500;
  border-radius: .2rem; cursor: pointer;
}
.nq-nav-btn:disabled { opacity: .35; cursor: not-allowed; }
.nq-prev {
  background: var(--md-default-bg-color);
  border: 1px solid var(--md-default-fg-color--lightest);
  color: var(--md-default-fg-color);
}
.nq-prev:hover:not(:disabled) { border-color: var(--md-primary-fg-color); }
.nq-next {
  background: var(--md-primary-fg-color);
  border: 1px solid var(--md-primary-fg-color);
  color: var(--md-primary-bg-color);
}
.nq-next:hover:not(:disabled) { opacity: .9; }

.nq-results { display: none; text-align: center; padding: 1.25rem 0; }
.nq-results.active { display: block; }
.nq-results-emoji { font-size: 1.75rem; margin-bottom: .3rem; }
.nq-results-score { font-size: 1.25rem; font-weight: 700; color: var(--md-primary-fg-color); }
.nq-results-text { font-size: .65rem; color: var(--md-default-fg-color--light); margin: .15rem 0 .6rem; }
.nq-results-stats { display: flex; justify-content: center; gap: 1.25rem; margin-bottom: .6rem; }
.nq-results-stat-val { font-size: .85rem; font-weight: 700; }
.nq-results-stat-val.correct { color: #2da44e; }
.nq-results-stat-val.wrong { color: #cf222e; }
.nq-results-stat-lbl { font-size: .55rem; color: var(--md-default-fg-color--light); }
.nq-results-msg { font-size: .65rem; margin-bottom: .8rem; }
.nq-reset {
  background: var(--md-primary-fg-color); color: var(--md-primary-bg-color);
  border: none; padding: .35rem .85rem; font: inherit; font-size: .65rem;
  font-weight: 500; border-radius: .2rem; cursor: pointer;
}
.nq-reset:hover { opacity: .9; }
</style>

<div class="nq-header">
  <div>Question <strong id="currQ">1</strong> of <strong>34</strong></div>
  <div><span class="nq-score-correct" id="correctCnt">0</span> correct · <span class="nq-score-wrong" id="wrongCnt">0</span> wrong</div>
</div>
<div id="quizCards"></div>
<div class="nq-results" id="results">
  <div class="nq-results-emoji" id="resEmoji">🎉</div>
  <div class="nq-results-score" id="resScore">0%</div>
  <div class="nq-results-text">Quiz Complete!</div>
  <div class="nq-results-stats">
    <div><div class="nq-results-stat-val correct" id="resCorrect">0</div><div class="nq-results-stat-lbl">Correct</div></div>
    <div><div class="nq-results-stat-val wrong" id="resWrong">0</div><div class="nq-results-stat-lbl">Wrong</div></div>
  </div>
  <div class="nq-results-msg" id="resMsg"></div>
  <button class="nq-reset" onclick="resetQuiz()">Try Again</button>
</div>
<div class="nq-nav" id="navBtns">
  <button class="nq-nav-btn nq-prev" id="prevBtn" onclick="go(-1)" disabled>← Previous</button>
  <button class="nq-nav-btn nq-next" id="nextBtn" onclick="go(1)">Next →</button>
</div>

<script>
const Q = [
  {q:"In a packet's journey across multiple routers, which addresses are typically rewritten at each hop, and which remain constant?",o:["Both IP and MAC addresses remain constant throughout the entire journey.","Only the destination MAC and destination IP addresses change at each hop.","Source and destination MAC addresses change at each hop, while source and destination IP addresses remain constant.","Source and destination IP addresses change at each hop, while source and destination MAC addresses remain constant."],a:2,h:"Routers operate at Layer 3. What gets rewritten when sending to the next hop?",link:"do-ip-addresses-change-as-packets-traverse-the-network",linkText:"IP vs MAC addresses in routing",exp:{c:"IP addresses stay constant (end-to-end identifiers). MAC addresses change at each hop (physical next-hop addressing).",w:["IP addresses remain constant, but MAC addresses must change at each hop for next-hop delivery.","Only MAC addresses change, not IP addresses.","Correct!","This is backwards. IP = end-to-end, MAC = hop-by-hop."]}},
  {q:"What is the primary innovation of Geneve that distinguishes it from VXLAN?",o:["It supports over 16 million logical networks using a 24-bit identifier.","It has a variable-length header that can carry custom metadata using Type-Length-Value (TLV) options.","It encapsulates Layer 2 frames within Layer 3 UDP packets to create an overlay network.","It uses BGP-EVPN as its primary control plane for discovering tunnel endpoints."],a:1,h:"VXLAN has a fixed 8-byte header. What does Geneve add for extensibility?",link:"what-is-geneve",linkText:"Geneve and TLV options",exp:{c:"Geneve's variable-length TLV options allow carrying security policies, identity, and telemetry inline with packets.",w:["Both VXLAN and Geneve support 16.7M networks. This isn't the distinction.","Correct!","Both VXLAN and Geneve do this. Not unique to Geneve.","Both can use BGP-EVPN. The control plane is independent of encapsulation."]}},
  {q:"A network administrator needs to create a subnet that can support approximately 60,000 devices. Which CIDR notation should they use?",o:["/24","/8","/16","/32"],a:2,h:"Formula: usable hosts = 2^(32 - prefix) - 2. What gives ~65,000 hosts?",link:"how-do-you-calculate-the-number-of-hosts-in-a-subnet",linkText:"Subnet calculation",exp:{c:"A /16 has 16 host bits: 2^16 = 65,536 addresses, 65,534 usable hosts.",w:["/24 = 254 hosts. Too small.","A /8 = 16M hosts. Way too large.","Correct!","/32 = single host. No room for devices."]}},
  {q:"What is the fundamental problem that workload identity standards like SPIFFE solve, which IP-based security cannot?",o:["Workload identity provides faster packet routing by using hardware offloading.","IP addresses are ephemeral and unreliable as identifiers for workloads that are frequently rescheduled, moved, or scaled.","IP-based security does not work across different cloud providers.","IP-based rules are difficult to write and require deep networking knowledge."],a:1,h:"Containers get new IPs on every restart. What happens to your firewall rules?",link:"what-is-the-evolution-of-workload-identity-and-what-problem-was-it-solving",linkText:"Workload identity evolution",exp:{c:"In dynamic environments, IPs change constantly. SPIFFE provides cryptographic identity that stays constant regardless of IP.",w:["Workload identity is about auth, not routing performance.","Correct!","IP-based security can work across clouds; the issue is IP ephemerality.","Difficulty isn't the core problem; unreliable identification is."]}},
  {q:"In Kubernetes, what is the role of an Endpoints or EndpointSlice object?",o:["It acts as a bridge connecting the stable Service IP to the dynamic, ephemeral IPs of the actual backing pods.","It defines the routing rules for external traffic coming into the cluster.","It enforces network policies to control which pods can communicate with each other.","It assigns a stable virtual IP address to a set of pods."],a:0,h:"Services have stable IPs, pods have ephemeral IPs. What tracks the mapping?",link:"why-do-we-need-endpoints",linkText:"Why Endpoints are needed",exp:{c:"Endpoints bridge Services (stable) and Pods (ephemeral). kube-proxy uses them to update routing rules.",w:["Correct!","That's Ingress, not Endpoints.","That's NetworkPolicy, not Endpoints.","That's the Service itself, not Endpoints."]}},
  {q:"Which of the following best describes the relationship between a service mesh and an overlay network like VXLAN?",o:["Service mesh operates at the application layer (L7) on top of the connectivity provided by an infrastructure-layer (L2/L3) overlay network.","Service mesh is a replacement for overlay networks, providing a more efficient way to route packets.","Overlay networks are a feature of service mesh used for creating encrypted tunnels between sidecars.","Service mesh and overlay networks are competing technologies that solve the same problem at different layers."],a:0,h:"VXLAN = L2/L3 packet routing. Service mesh = L7 request routing. How do they relate?",link:"how-does-service-mesh-relate-to-overlay-networks",linkText:"Service mesh and overlays",exp:{c:"They're complementary. Overlay provides packet routing; service mesh adds L7 capabilities on top.",w:["Correct!","Service mesh requires overlay/CNI for packet routing. It doesn't replace it.","Overlay is infrastructure-layer, independent of service mesh.","They complement, not compete. Different layers, different purposes."]}},
  {q:"Why does VXLAN use UDP for its transport protocol instead of TCP?",o:["Because UDP has lower overhead and avoids the 'TCP-in-TCP' problem, as reliability is already handled by the inner packet.","To ensure guaranteed, in-order delivery of the encapsulated frames across the physical network.","Because UDP headers have a dedicated field for the Virtual Network Identifier (VNI), while TCP does not.","To leverage TCP's advanced congestion control mechanisms to avoid overwhelming the underlay network."],a:0,h:"Inner packets often contain TCP. What happens with TCP-in-TCP?",link:"why-does-vxlan-use-udp",linkText:"Why VXLAN uses UDP",exp:{c:"UDP avoids double reliability overhead and TCP-in-TCP retransmission storms. Inner TCP handles reliability.",w:["Correct!","That describes TCP. UDP has no delivery guarantees.","VNI is in the VXLAN header, not UDP header.","That describes TCP, which VXLAN avoids."]}},
  {q:"What is the primary function of a veth pair in Linux container networking?",o:["To assign an IP address to a container from a predefined range.","To act as a virtual switch connecting multiple containers on the same host.","To enforce resource limits, such as network bandwidth, for a container.","To provide a virtual, point-to-point Ethernet cable that connects a container's network namespace to the host's root network namespace."],a:3,h:"Think of it as a virtual cable with two ends. Where do the ends go?",link:"what-is-a-veth-pair",linkText:"What is a veth pair",exp:{c:"A veth pair is a virtual cable: one end in the container's namespace, one in the host's. Packets in one end appear at the other.",w:["That's IPAM, not veth pairs.","That's a bridge. Veth is point-to-point.","That's cgroups, not veth pairs.","Correct!"]}},
  {q:"A CNI plugin like Calico is configured in 'native routing' mode using BGP. What does this configuration achieve?",o:["It allows the underlying physical network to learn pod IP routes directly, eliminating the need for an overlay network.","It encapsulates all pod-to-pod traffic in VXLAN tunnels for maximum compatibility.","It creates a mesh of tunnels between every node in the cluster for resilient communication.","It enforces all network policies using eBPF for higher performance."],a:0,h:"BGP advertises routes. What if your physical routers know about pod CIDRs?",link:"how-does-cilium-use-bgp",linkText:"CNI plugins and BGP",exp:{c:"With BGP, physical routers learn pod routes directly. No overlay encapsulation needed.",w:["Correct!","That's overlay mode, the opposite of native routing.","BGP advertises routes, doesn't create tunnels.","eBPF is separate from BGP."]}},
  {q:"What is the purpose of the Time To Live (TTL) field in an IP packet header?",o:["To prioritize high-importance packets over lower-importance ones.","To signal to the destination host how long to keep the connection open.","To prevent packets from circulating indefinitely in the network in the event of a routing loop.","To measure the round-trip latency between the source and destination hosts."],a:2,h:"What stops a packet from bouncing between routers forever?",link:"what-is-ttl-and-why-is-it-important",linkText:"TTL explained",exp:{c:"TTL decrements at each hop. At 0, the packet is dropped, preventing infinite loops.",w:["That's QoS/DSCP, not TTL.","TTL is not about connection duration.","Correct!","Latency uses timestamps/ICMP, not TTL."]}},
  {q:"Which Linux kernel feature provides Layer 4 load balancing and is often used by kube-proxy in large clusters for better performance than iptables?",o:["Network Namespaces","cgroups (control groups)","eBPF (extended Berkeley Packet Filter)","IPVS (IP Virtual Server)"],a:3,h:"kube-proxy has iptables mode and another high-performance mode using hash tables.",link:"what-is-linux-ipvs-ip-virtual-server",linkText:"IPVS explained",exp:{c:"IPVS uses hash tables for O(1) lookup vs iptables O(n) chain traversal. Much faster for large clusters.",w:["Namespaces provide isolation, not load balancing.","cgroups limit resources, not load balancing.","eBPF can do this (Cilium), but kube-proxy uses IPVS.","Correct!"]}},
  {q:"In the OSI model, which layer is responsible for routing packets between different networks using IP addresses?",o:["Layer 3: Network Layer","Layer 7: Application Layer","Layer 2: Data Link Layer","Layer 4: Transport Layer"],a:0,h:"L2 = MAC/switches. L3 = IP/routers. L4 = ports/TCP/UDP.",link:"what-is-layer-3-about",linkText:"Layer 3 explained",exp:{c:"Layer 3 (Network) handles IP addressing and routing between networks. This is where routers operate.",w:["Correct!","L7 is HTTP, DNS, etc.","L2 is MAC addresses and switches.","L4 is TCP/UDP ports."]}},
  {q:"What was a major limitation of VLANs that led to the development of technologies like VXLAN for cloud environments?",o:["VLANs introduced too much latency by adding a large header to every Ethernet frame.","The 12-bit VLAN ID field limited the number of possible networks to 4,094, which was insufficient for large multi-tenant clouds.","VLANs could not be configured to work with modern fiber optic cables.","VLANs required the use of the UDP protocol, which was considered unreliable for enterprise traffic."],a:1,h:"Cloud providers need thousands of tenant networks. 12 bits = how many networks?",link:"what-were-the-constraints-of-vlans",linkText:"VLAN constraints",exp:{c:"12-bit VLAN ID = 4,094 networks max. Cloud needs more. VXLAN's 24-bit VNI = 16.7M networks.",w:["VLAN tag is only 4 bytes. Negligible overhead.","Correct!","VLANs work fine with fiber.","VLANs don't use UDP. They're L2."]}},
  {q:"When troubleshooting a non-functional Kubernetes Service, you find that the Service exists and has a ClusterIP, but `kubectl get endpoints` returns an empty list. What is the most likely cause?",o:["A NetworkPolicy is blocking traffic between kube-proxy and the pods.","CoreDNS is not configured correctly, so the Service name cannot be resolved.","The CNI plugin is not running on the nodes, so pods have no IP addresses.","The labels on the pods do not match the selector defined in the Service manifest."],a:3,h:"Endpoints are populated when pods match the Service's selector. What if nothing matches?",link:"what-do-i-do-if-a-service-is-not-working",linkText:"Troubleshooting Services",exp:{c:"Empty Endpoints = no pods match the selector. Check pod labels vs Service selector.",w:["NetworkPolicy affects traffic flow, not Endpoint population.","DNS issues don't affect Endpoint creation.","If pods had no IPs they wouldn't be Running.","Correct!"]}},
  {q:"How does a service mesh like Istio enforce identity-based authorization policies in Kubernetes?",o:["The CNI plugin inspects each packet's source IP and compares it against allowed IPs.","The sidecar proxy (e.g., Envoy) intercepts all traffic, verifies the cryptographic identity (SPIFFE SVID), and allows or denies based on policy.","The Kubernetes API server intercepts each request and verifies the pod's service account.","The kube-proxy component applies authorization rules using IPVS or iptables."],a:1,h:"Service mesh sidecars intercept traffic and can verify certificates. What do they use for identity?",link:"where-do-i-create-identity-based-security-rules-and-who-enforces-them-in-kubernetes",linkText:"Identity-based authorization",exp:{c:"Sidecars perform mTLS, verify SPIFFE SVIDs, and enforce AuthorizationPolicy. All transparent to apps.",w:["That's IP-based security (NetworkPolicy), not identity-based.","Correct!","API server handles resource operations, not service-to-service traffic.","kube-proxy does routing, not authorization."]}},
  {q:"Which of the following is NOT one of the three fundamental requirements of the Kubernetes networking model?",o:["Agents on a node (like kubelet) can communicate with all pods on that node.","Every pod gets its own unique IP address.","All pods can communicate with all other pods across all nodes without NAT.","All traffic between pods must be encrypted by default."],a:3,h:"K8s networking has 3 requirements about IPs and connectivity. Encryption is not one of them.",link:"what-are-the-three-fundamental-requirements-of-kubernetes-networking",linkText:"K8s networking requirements",exp:{c:"The 3 requirements: (1) IP per pod, (2) pod-to-pod without NAT, (3) agents can reach pods. Encryption is NOT required by default.",w:["This IS a requirement.","This IS a requirement.","This IS a requirement.","Correct! Encryption is NOT a K8s networking requirement."]}},
  {q:"When a host uses ARP, what information is it broadcasting to the local network, and what information is it expecting in return?",o:["It broadcasts its own IP address and requests a MAC address from a DHCP server.","It broadcasts its own MAC address and requests the MAC address of the default gateway.","It broadcasts a destination MAC address and requests the corresponding IP address.","It broadcasts a destination IP address and requests the corresponding MAC address."],a:3,h:"ARP = Address Resolution Protocol. It resolves IP → MAC.",link:"how-does-arp-work",linkText:"How ARP works",exp:{c:"ARP broadcasts 'Who has IP X?' and expects the response 'IP X is at MAC Y.'",w:["That's DHCP, not ARP.","ARP finds MAC for a given IP, not the other way around.","ARP resolves IP→MAC, not MAC→IP.","Correct!"]}},
  {q:"You are experiencing an issue where small network payloads between nodes work fine, but larger HTTPS requests fail or time out. This is a classic symptom of what kind of problem?",o:["Incorrect BGP peer configuration between nodes.","An MTU mismatch between the overlay and the underlay networks.","The CNI plugin's IPAM has exhausted its IP address pool.","A DNS resolution failure for cross-node services."],a:1,h:"VXLAN adds ~50 bytes overhead. If underlay MTU is 1500 and pod sends 1500-byte packets...",link:"how-do-i-avoid-vxlan-fragmentation",linkText:"MTU issues",exp:{c:"Classic MTU mismatch: large packets + overlay overhead exceed underlay MTU, causing fragmentation/drops.",w:["BGP issues would break all traffic, not just large payloads.","Correct!","IPAM exhaustion prevents new pods from getting IPs.","DNS issues affect all requests, not size-dependent."]}},
  {q:"What is the key difference between how cgroups and network namespaces contribute to containerization?",o:["Network namespaces provide network isolation, while cgroups provide resource limitation and accounting.","Both are networking primitives that work together to create the overlay network.","cgroups provide network connectivity, while namespaces provide security policies.","cgroups provide an isolated network stack, while namespaces limit resource usage."],a:0,h:"One creates isolation (separate network stack). One limits resources (CPU, memory, bandwidth).",link:"what-are-cgroups-and-are-they-a-networking-primitive",linkText:"cgroups vs namespaces",exp:{c:"Namespaces = isolation (separate interfaces, routes). cgroups = resource limits (CPU, memory, bandwidth).",w:["Correct!","cgroups aren't networking primitives.","This is backwards.","This is backwards."]}},
  {q:"Which of the following problems is a service mesh specifically designed to solve at the application layer (L7)?",o:["Segmenting a physical network into multiple logical broadcast domains.","Managing service discovery, traffic management, and observability transparently to the application.","Ensuring packets can be routed between different physical hosts.","Assigning a unique IP address to every container instance."],a:1,h:"Before service mesh, every service implemented its own retries, TLS, metrics. Service mesh centralizes this at L7.",link:"what-problem-does-service-mesh-solve",linkText:"Problems service mesh solves",exp:{c:"Service mesh handles L7 concerns: service discovery, retries, timeouts, mTLS, tracing—all transparent to apps.",w:["That's VLANs (L2).","Correct!","That's routers/overlay (L3).","That's CNI/IPAM."]}},
  {q:"In Kubernetes, a NetworkPolicy allows traffic from `app: frontend` to `app: backend` on port 8080. Who enforces this rule?",o:["The Kubernetes API server.","The CoreDNS server.","The CNI plugin (e.g., Cilium or Calico).","The sidecar proxy injected by a service mesh."],a:2,h:"NetworkPolicy is L3/L4 enforcement. What component programs iptables/eBPF rules at the kernel level?",link:"how-are-network-policies-implemented",linkText:"NetworkPolicy implementation",exp:{c:"CNI plugins implement NetworkPolicy by programming kernel-level rules (iptables/eBPF).",w:["API server stores policies, doesn't enforce them.","CoreDNS does DNS, not network policies.","Correct!","Service mesh does L7 authorization, not L3/L4 NetworkPolicy."]}},
  {q:"How does NAT allow multiple devices on a private network to share a single public IP address?",o:["It assigns a unique public IP from a pool to each device.","It rewrites the source IP and port of outgoing packets and maintains a state table to route replies back.","It broadcasts a request to the public internet to find a route.","It encapsulates packets from private IPs inside a packet with the public IP."],a:1,h:"NAT rewrites addresses and uses port numbers to track connections.",link:"what-is-nat-and-how-does-it-work",linkText:"How NAT works",exp:{c:"NAT rewrites source IP/port, maintains a mapping table, and uses it to forward replies to the correct internal device.",w:["That's NAT pools, less common.","Correct!","NAT doesn't broadcast.","That's encapsulation (like VXLAN), not NAT."]}},
  {q:"Which statement accurately compares TCP and UDP?",o:["UDP establishes a three-way handshake for reliability, while TCP is 'fire and forget'.","TCP is connectionless with low overhead for video streaming, while UDP guarantees delivery.","Both provide the same reliability, just different use cases.","TCP provides reliable, ordered delivery (like registered mail), while UDP is connectionless with no guarantees (like regular mail)."],a:3,h:"One has handshakes and retransmissions. One just sends data and hopes.",link:"whats-the-difference-between-tcp-and-udp",linkText:"TCP vs UDP",exp:{c:"TCP = connection-oriented, reliable, ordered. UDP = connectionless, no guarantees, faster.",w:["Backwards. TCP does handshake.","Backwards. TCP is connection-oriented.","They have different reliability levels.","Correct!"]}},
  {q:"What distinguishes a Layer 2 switch from a Layer 3 router?",o:["A switch operates at the physical layer; a router at the data link layer.","A switch forwards traffic based on MAC addresses within a local network; a router forwards based on IP addresses between networks.","A switch uses routing tables; a router uses ARP tables.","A router modifies IP addresses; a switch does not."],a:1,h:"Switch = L2/MAC/local. Router = L3/IP/between networks.",link:"what-is-a-switch-and-how-does-it-work",linkText:"Switch vs router",exp:{c:"Switches use MAC addresses for local delivery. Routers use IP addresses to forward between networks.",w:["Wrong layers.","Correct!","Backwards.","Routers don't modify IPs (except NAT)."]}},
  {q:"How does IPv6 solve IPv4 address exhaustion?",o:["By introducing more efficient subnetting.","By relying on NAT to share addresses.","By using a 64-bit address space.","By expanding the address space to 128 bits."],a:3,h:"IPv4 = 32 bits = 4.3B addresses. IPv6 went much bigger.",link:"what-is-an-ip-address",linkText:"IPv4 vs IPv6",exp:{c:"IPv6 uses 128-bit addresses: 2^128 = 340 undecillion addresses. Practically unlimited.",w:["Subnetting helps but doesn't solve exhaustion.","NAT is a workaround, not a solution.","64 bits would be 2^64. IPv6 uses 128 bits.","Correct!"]}},
  {q:"What is the role of kube-proxy in iptables mode?",o:["It acts as the DNS server for the cluster.","It runs on every node and maintains iptables rules that map Service ClusterIPs to pod IPs.","It runs as a sidecar proxy in each pod.","It is a CNI plugin that assigns IP addresses to pods."],a:1,h:"ClusterIP → Pod IPs translation. Who programs the iptables DNAT rules?",link:"what-is-kube-proxy-and-what-are-its-modes",linkText:"kube-proxy explained",exp:{c:"kube-proxy watches Services/Endpoints and programs iptables rules to route ClusterIP traffic to pods.",w:["That's CoreDNS.","Correct!","That's service mesh sidecars.","That's CNI plugins."]}},
  {q:"Which security task is best handled by NetworkPolicy rather than service mesh authorization?",o:["Enforcing that only 'admin' users can access /metrics.","Creating a default-deny rule between 'dev' and 'prod' namespaces.","Ensuring mTLS encryption between services.","Allowing GET but denying POST to a service."],a:1,h:"NetworkPolicy = L3/L4 (namespaces, ports). Service mesh = L7 (HTTP methods, paths, users).",link:"how-are-network-policies-implemented",linkText:"NetworkPolicy vs service mesh",exp:{c:"Namespace isolation is L3/L4—perfect for NetworkPolicy. No HTTP parsing needed.",w:["User roles need L7 inspection (service mesh).","Correct!","mTLS is service mesh.","HTTP methods need L7 inspection (service mesh)."]}},
  {q:"Which statement accurately represents the relationship between Kubernetes Ingress and Service?",o:["Ingress replaces Services by routing directly to pods.","Ingress routes external HTTP/S traffic to Services within the cluster.","Services of type 'Ingress' handle L4 load balancing.","Ingress is a type of Service that exposes pods to the internet."],a:1,h:"External traffic → Ingress → Service → Pods. Ingress uses Services.",link:"what-is-ingress-and-how-does-it-relate-to-services",linkText:"Ingress explained",exp:{c:"Ingress provides L7 routing (host/path) for external HTTP/S traffic, forwarding to Services.",w:["Ingress uses Services, doesn't replace them.","Correct!","There's no 'Ingress' Service type.","Ingress is a separate resource, not a Service type."]}},
  {q:"When are you told you probably don't need a service mesh?",o:["Your org mandates zero-trust with mTLS everywhere.","You have 10+ microservices and can't trace requests across them.","You're building a simple two-tier app with frontend and database.","Teams are implementing inconsistent retry logic in every service."],a:2,h:"Service mesh adds complexity. Simple apps don't need its features.",link:"do-i-need-service-mesh",linkText:"Do I need service mesh?",exp:{c:"A simple two-tier app doesn't benefit from service mesh. No complex service communication to manage.",w:["mTLS mandate = you DO need service mesh.","Tracing 10+ services = you DO need service mesh.","Correct!","Centralizing retry logic = you DO need service mesh."]}},
  {q:"What is the key difference between how a Layer 2 switch and Layer 3 router make forwarding decisions?",o:["A switch broadcasts all traffic; a router sends to specific ports.","A switch modifies IP headers; a router modifies Ethernet frames.","A switch uses IP addresses; a router uses MAC addresses.","A switch forwards within a local network; a router forwards between networks."],a:3,h:"Switch = local/MAC. Router = between networks/IP.",link:"what-is-a-router-and-how-does-routing-work",linkText:"Switch vs router",exp:{c:"Switches forward within a L2 segment using MAC. Routers forward between L3 networks using IP.",w:["Switches learn MACs and forward to specific ports.","Backwards.","Backwards.","Correct!"]}},
  {q:"How does CoreDNS facilitate service discovery in Kubernetes?",o:["It watches Service objects and creates DNS A records mapping service names to ClusterIPs.","It intercepts pod traffic and rewrites destination IPs.","It maintains a list of all pod IPs.","It injects environment variables with service IPs into pods."],a:0,h:"Apps query DNS for 'backend.default.svc'. CoreDNS returns the ClusterIP.",link:"what-is-coredns",linkText:"CoreDNS explained",exp:{c:"CoreDNS watches Services and creates DNS records. Apps query DNS, get ClusterIP, kube-proxy routes to pods.",w:["Correct!","CoreDNS only does DNS, not traffic interception.","CoreDNS returns ClusterIPs (unless headless), not pod IPs directly.","Env vars are a legacy feature, not CoreDNS."]}},
  {q:"What is the primary purpose of the TCP three-way handshake?",o:["To determine the shortest network path.","To negotiate IP addresses.","To allow both sides to agree on initial sequence numbers and verify two-way communication.","To encrypt the connection."],a:2,h:"Before reliable data transfer, both sides need to agree on sequence numbers.",link:"what-is-the-tcp-three-way-handshake",linkText:"TCP handshake",exp:{c:"The handshake establishes sequence numbers and confirms both sides can send and receive.",w:["Path is determined by routing, not TCP.","IPs are known before the handshake.","Correct!","Encryption (TLS) happens after TCP handshake."]}},
  {q:"How does traffic from an application container get intercepted by the service mesh sidecar proxy?",o:["The sidecar registers with CoreDNS to receive traffic.","The CNI plugin routes all pod traffic to the sidecar first.","The application is recompiled with a special library.","iptables rules in the pod's network namespace redirect all traffic to the sidecar."],a:3,h:"Service mesh is transparent—no app changes. An init container sets up iptables rules.",link:"how-does-service-mesh-integrate-with-kubernetes-networking",linkText:"Sidecar traffic interception",exp:{c:"An init container configures iptables rules to capture all pod traffic and redirect it to the sidecar.",w:["CoreDNS only does DNS.","CNI doesn't route to sidecars.","Service mesh is transparent—no code changes.","Correct!"]}},
  {q:"What is the primary role of BGP on the public internet?",o:["To exchange routing information between different autonomous systems (networks).","To encrypt traffic between networks.","To assign public IP addresses.","To resolve domain names to IP addresses."],a:0,h:"BGP is how ISPs and networks learn about each other's IP ranges.",link:"what-is-bgp-border-gateway-protocol",linkText:"BGP explained",exp:{c:"BGP allows autonomous systems to advertise their IP ranges and learn paths to reach any IP on the internet.",w:["Correct!","BGP doesn't encrypt.","IP assignment is IANA/RIRs.","DNS resolves names. BGP handles routing."]}}
];

let cur = 0, correct = 0, wrong = 0, answered = Array(Q.length).fill(null);

function render() {
  document.getElementById('quizCards').innerHTML = Q.map((q, i) => `
    <div class="nq-card${i===0?' active':''}" data-i="${i}">
      <div class="nq-question">${q.q}</div>
      <div class="nq-options">${q.o.map((opt, j) => `
        <div class="nq-opt" id="opt-${i}-${j}">
          <input type="radio" name="q${i}" id="r${i}-${j}" onchange="pick(${i},${j})">
          <label for="r${i}-${j}">
            <span class="nq-opt-letter">${String.fromCharCode(65+j)}.</span>
            <span class="nq-opt-text">${opt}</span>
          </label>
          <div class="nq-opt-exp" id="exp-${i}-${j}"></div>
        </div>`).join('')}
      </div>
      <div class="nq-hint">
        <button class="nq-hint-btn" onclick="hint(${i})">💡 Hint</button>
        <div class="nq-hint-text" id="ht-${i}">${q.h}</div>
      </div>
    </div>
  `).join('');
  upd();
}

function hint(i) { document.getElementById('ht-'+i).classList.toggle('visible'); }

function pick(i, j) {
  if (answered[i] !== null) return;
  const q = Q[i], ok = j === q.a;
  answered[i] = { j, ok };
  if (ok) correct++; else wrong++;

  q.o.forEach((_, k) => {
    const el = document.getElementById('opt-'+i+'-'+k);
    el.classList.add('disabled');
    el.querySelector('input').disabled = true;
  });

  const sel = document.getElementById('opt-'+i+'-'+j);
  const cor = document.getElementById('opt-'+i+'-'+q.a);

  if (ok) {
    sel.classList.add('correct');
    const exp = document.getElementById('exp-'+i+'-'+j);
    exp.innerHTML = `<div class="nq-opt-exp-title">✓ Correct</div>${q.exp.c}`;
    exp.classList.add('correct','visible');
  } else {
    sel.classList.add('wrong');
    cor.classList.add('correct');
    const expWrong = document.getElementById('exp-'+i+'-'+j);
    expWrong.innerHTML = `<div class="nq-opt-exp-title">✗ Incorrect</div>${q.exp.w[j]}`;
    expWrong.classList.add('wrong','visible');
    const expCor = document.getElementById('exp-'+i+'-'+q.a);
    expCor.innerHTML = `<div class="nq-opt-exp-title">✓ Correct answer</div>${q.exp.c}`;
    expCor.classList.add('correct','visible');
  }
  upd();
}

function upd() {
  document.getElementById('currQ').textContent = cur + 1;
  document.getElementById('correctCnt').textContent = correct;
  document.getElementById('wrongCnt').textContent = wrong;
  document.getElementById('prevBtn').disabled = cur === 0;
  document.getElementById('nextBtn').textContent = cur === Q.length - 1 ? 'Finish →' : 'Next →';
}

function go(d) {
  if (d === 1 && cur === Q.length - 1) { showRes(); return; }
  document.querySelectorAll('.nq-card').forEach(c => c.classList.remove('active'));
  cur = Math.max(0, Math.min(Q.length - 1, cur + d));
  document.querySelector('.nq-card[data-i="'+cur+'"]').classList.add('active');
  upd();
}

function showRes() {
  document.querySelectorAll('.nq-card').forEach(c => c.classList.remove('active'));
  document.getElementById('navBtns').style.display = 'none';
  document.querySelector('.nq-header').style.display = 'none';

  const pct = Math.round(correct / Q.length * 100);
  document.getElementById('resScore').textContent = pct + '%';
  document.getElementById('resCorrect').textContent = correct;
  document.getElementById('resWrong').textContent = wrong;

  let emoji, msg;
  if (pct >= 90) { emoji = '🏆'; msg = 'Outstanding! Excellent networking knowledge.'; }
  else if (pct >= 70) { emoji = '🎉'; msg = 'Great job! Solid understanding.'; }
  else if (pct >= 50) { emoji = '📚'; msg = 'Good effort! Review the topics you missed.'; }
  else { emoji = '💪'; msg = 'Keep learning! The guide covers all these topics.'; }

  document.getElementById('resEmoji').textContent = emoji;
  document.getElementById('resMsg').textContent = msg;
  document.getElementById('results').classList.add('active');
}

function resetQuiz() {
  cur = 0; correct = 0; wrong = 0; answered = Array(Q.length).fill(null);
  document.getElementById('results').classList.remove('active');
  document.getElementById('navBtns').style.display = 'flex';
  document.querySelector('.nq-header').style.display = 'flex';
  render();
}

document.addEventListener('DOMContentLoaded', render);
if (document.readyState !== 'loading') render();
</script>

---

*Based on [Networking in a Hurry: From ARP to Service Mesh](networking-in-a-hurry.md)*